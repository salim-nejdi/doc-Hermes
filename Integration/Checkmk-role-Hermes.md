# Un rôle Checkmk dédié à Hermes : trois profils de confiance

Ce document décrit comment donner à Hermes un compte **dédié** dans Checkmk pour qu'il puisse
corriger les sondes et les checks mal réglés — typiquement : « le seuil configuré est resté au
défaut, la machine ne l'a jamais atteint et le service clignote en WARNING/CRIT sans raison ».

Trois niveaux de confiance croissants :

1. **lecture seule** — il analyse et propose, il ne modifie rien ;
2. **approbation manuelle** — il prépare le correctif, un humain l'active ;
3. **full-automatique** — il corrige, active, mesure, et revient en arrière si c'est pire.

Le levier technique qui rend ces trois profils possibles : dans Checkmk, **écrire une règle** et
**activer la configuration** sont deux permissions distinctes. Retirer la seconde transforme
l'agent en proposeur sans lui retirer la capacité technique de préparer le correctif.

> **Statut** — document de conception, tenu à jour au fil de la mise en place et des tests.
> Les chemins d'API et les identifiants de permission cités proviennent du code source de
> Checkmk (branche `master`, relevé du 16/09/2026) : les points vérifiés sont marqués ✅,
> ceux qui restent à valider sur un vrai site sont marqués ⚠️.

---

## 1. Pourquoi un rôle dédié

Réutiliser un compte administrateur pour l'agent revient à lui donner les clés de la
supervision entière (utilisateurs, notification, sites, mots de passe, réglages globaux).
Un rôle dédié borne ce qu'il peut casser :

- **portée** — le rôle peut être limité à un dossier de la configuration, ou à un contact group ;
- **traçabilité** — tout ce qu'il écrit apparaît dans le journal d'audit sous son nom ;
- **réversibilité** — un correctif = une règle identifiable (préfixe `[hermes]`) qu'on supprime
  pour revenir à l'état précédent.

## 2. Pré-requis

- Un site Checkmk avec l'API REST (≥ 2.3) et, si l'agent doit écrire, les endpoints
  `rule` + `activation_run` (voir §8).
- **Un utilisateur d'automation dédié** plutôt qu'un compte humain : dans Setup, la création
  d'un utilisateur de type *Automation user* génère un secret qui sert d'identifiant d'API.
  Ce compte n'a pas d'accès à l'interface web.
- Si l'authentification à deux facteurs est imposée par défaut, elle doit rester
  **non appliquée** à ces rôles : un compte d'automation sans second facteur est
  inutilisable autrement.
- Authentification des appels : en-tête `Authorization: Bearer <utilisateur> <secret>`
  (ou HTTP Basic avec le secret en mot de passe) — comportement documenté dans
  `docs.checkmk.com/latest/en/rest_api.html`.

## 3. Les permissions utiles : libellé GUI ↔ identifiant interne

Identifiants relevés dans `cmk/gui/wato/_permissions.py` (branche `master`) ✅.

| Libellé dans l'interface | Identifiant | Ce que ça ouvre |
|---|---|---|
| Use Setup | `wato.use` | accès au module Setup |
| Read access to all hosts and folders | `wato.see_all_folders` | sinon, seuls les dossiers des contact groups de l'utilisateur |
| Write access to all hosts and folders | `wato.all_folders` | sinon, écriture limitée aux dossiers de ses contact groups |
| Read access to all modules | `wato.seeall` | sinon, les modules restent filtrés par contact group |
| See all host and services | `general.see_all` | visibilité de tous les hôtes et services supervisés |
| Rule sets | `wato.rulesets` | accès aux règles ; **seul**, insuffisant pour écrire |
| Make changes, perform actions | `wato.edit` | toute modification ; **indispensable** pour créer/modifier une règle |
| Activate configuration | `wato.activate` | appliquer les modifications en attente à la supervision |
| Activate foreign changes | `wato.activateforeign` | activer aussi les modifications des autres utilisateurs |
| Audit log | `wato.auditlog` | historique des modifications |
| Revert changes | `wato.discard` | annuler **ses propres** modifications en attente |
| Revert foreign changes | `wato.discardforeign` | annuler celles des autres |
| Write access to all passwords | `wato.edit_all_passwords` | mots de passe stockés, tous contact groups |
| Add or modify executables | `wato.add_or_modify_executables` | commandes brutes exécutées sur les hôtes — **à ne pas donner** |

> Attention : `Revert changes` (`wato.discard`) agit sur les **modifications en attente**
> (pending changes), pas sur une règle déjà activée. Annuler un correctif déjà actif se fait
> en supprimant la règle puis en activant.

### Ce que l'API exige réellement

Vérifié dans les endpoints OpenAPI de Checkmk (branche `master`) ✅ :

- **lire** une règle ou un jeu de règles → `wato.rulesets` **seul**
  (`GET /objects/rule/{id}`, `GET /objects/ruleset/{nom}`) ; la recherche « tous jeux
  confondus » ajoute `wato.all_folders` ;
- **créer / modifier / déplacer / supprimer** une règle → `wato.edit` **et** `wato.rulesets`,
  plus le droit d'écriture sur le dossier visé ;
- **activer** la configuration → `wato.activate` ;
- lire les **modifications en attente** par l'API → `wato.activate` également (particularité :
  l'interface les montre à tout utilisateur ayant accès à Setup, l'API non).

C'est cette dernière séparation qui structure les trois profils ci-dessous.

---

## 4. La liste des cases à cocher, rôle par rôle

Trois rôles à créer dans **Setup ▸ Users ▸ Roles & permissions**, chacun basé sur le rôle *user*
(jamais *admin*). Règle de lecture : **tout ce qui n'est pas listé ici reste décoché.**

### `hermes-read` — lecture seule

| Section | Case à cocher | Identifiant |
|---|---|---|
| General | See all host and services | `general.see_all` |
| Setup | Use Setup | `wato.use` |
| Setup | Read access to all modules | `wato.seeall` |
| Setup | Read access to all hosts and folders | `wato.see_all_folders` |
| Setup | Rule sets | `wato.rulesets` |

`Rule sets` est un droit de **lecture** : il ouvre la consultation des règles, dans l'interface
comme par l'API. Toute écriture sur une règle exige en plus `Make changes, perform actions`
(`wato.edit`), qui reste décochée ici — c'est la garantie de ce profil (démonstration
opération par opération en §11).

Rien d'autre dans *Setup* : en particulier, aucune case d'écriture et aucune activation.

> **Et *Audit log* (`wato.auditlog`) ?** Elle n'est pas là, à dessein. C'est bien une permission de
> **lecture** (consulter l'historique des modifications), mais elle ne sert pas à lire les sondes :
> sa place naturelle est dans `hermes-ops`, qui écrit et doit pouvoir vérifier ce qu'il a changé.
> L'ajouter ici n'ouvre aucun droit d'écriture — à cocher si on veut que l'agent, en lecture seule,
> sache répondre à « qu'est-ce qui a changé depuis hier ? ».
>
> **Et la section *Topics* ?** Elle ne fait partie d'aucun profil, ni en lecture ni en écriture :
> ce n'est pas un droit sur la supervision, seulement l'affichage des thèmes dans le menu (détail
> plus bas dans cette section). La cocher ou non ne change rien à ce que le rôle peut faire.

### `hermes-ops` — il prépare le correctif, l'humain active

Les 5 cases ci-dessus, **plus** :

| Section | Case à cocher | Identifiant |
|---|---|---|
| Setup | Make changes, perform actions | `wato.edit` — c'est elle qui autorise l'écriture des règles |
| Setup | Audit log | `wato.auditlog` |
| Setup | Revert changes | `wato.discard` |
| Setup | Write access to all hosts and folders | `wato.all_folders` — **seulement** si la portée est « tous les hôtes » ; sinon laisser décoché et travailler dans un dossier |
| Setup | Write access to all passwords | `wato.edit_all_passwords` — facultatif, utile seulement pour un check à mot de passe stocké |

Et la seule case qui fait vraiment la différence pour ce profil :

| Section | Case à LAISSER DÉCOCHÉE | Identifiant |
|---|---|---|
| Setup | Activate configuration | `wato.activate` |

### `hermes-auto` — boucle fermée

Les 5 cases de `hermes-read`, les 5 nécessaires de `hermes-ops`, **plus** :

| Section | Case à cocher | Identifiant |
|---|---|---|
| Setup | Activate configuration | `wato.activate` |

`Activate foreign changes` et `Revert foreign changes` restent décochés dans les trois rôles :
l'agent n'active et n'annule que ses propres modifications.

### À ne cocher dans aucun des trois rôles

| Section | Case | Identifiant | Pourquoi |
|---|---|---|---|
| Setup | Add or modify executables | `wato.add_or_modify_executables` | exécution de commandes arbitraires sur les hôtes supervisés |
| Setup | Notification configuration | `wato.notifications` | l'agent pourrait modifier ses propres alertes |
| Setup | Global settings | `wato.global` | réglages du site entier |
| Setup | User management | `wato.users` | comprendrait la modification de son propre rôle |
| Setup | Host management | `wato.hosts` | créer ou supprimer des hôtes supervisés |
| Setup | Host & service groups | `wato.groups` | structure de la supervision |
| Setup | Archive audit log | `wato.clear_auditlog` | effacerait ses propres traces |
| Setup | Activate foreign changes | `wato.activateforeign` | embarquerait les modifications des autres utilisateurs |
| Setup | Revert foreign changes | `wato.discardforeign` | idem, en suppression |
| General | Edit / force / delete foreign `…`, Modify built-in `…` | `general.edit_*`, `general.force_*` | édition des objets créés par d'autres utilisateurs |
| Modules | Global settings, User management, Roles, Site management, Backup & restore, Event Console, BI, NagVis, snapshots | — | hors périmètre |

### La section « Topics » : visibilité seule, sans effet sur les droits d'écriture

La liste des thèmes — `Applications (applications)`, `IT infrastructure efficiency
(it_efficiency)`, `Monitoring (monitoring)` … — n'est pas écrite à la main dans le rôle éditeur :
elle est **générée par les pages** de Checkmk, une entrée par thème de tableau de bord, avec
l'identifiant `pagetype_topic.<nom>` (mécanisme vérifié dans `cmk/gui/pagetypes/_core.py` ✅).

Cocher une de ces cases décide seulement si le thème — et les tableaux de bord publiés dedans —
apparaît dans le menu de Hermes. Cela n'ouvre **aucun** droit de modification : ni Setup, ni
règle, ni hôte, ni activation. **Tout mettre à Oui est donc sans risque.** Si la première
version de cette page disait de ne pas les cocher, c'était pour garder le rôle minimal, pas par
sécurité. Pour un menu plus propre, on peut ne garder que `Monitoring`, `Overview` et
`Problems` ; c'est purement cosmétique.

Même mécanique pour la section *Dashboards* (`checkmk`, `simple_problems`, `main` ) : visibilité
des tableaux de bord nommés. Et les cases `… dashboards` de la section *General* ne concernent
que la création ou l'édition des tableaux de bord **des utilisateurs**, sans effet sur la
supervision des hôtes.

## 5. Profil 1 — lecture seule (`hermes-read`)

**Permissions** : Use Setup, Read access to all hosts and folders, Read access to all modules,
See all host and services. Aucun droit d'écriture, aucune activation.

**Ce qu'il fait** : lire l'état courant, l'historique des alertes et la configuration des
règles ; produire un diagnostic chiffré (« ce service est en WARNING depuis 6 jours, le seuil
est resté au défaut, les mesures réelles plafonnent à 60 % de ce seuil »).

**Exemple** — un service d'alerte de consommation : le seuil par défaut vise une valeur que le
matériel n'atteint jamais, le service oscille donc entre OK et WARNING au moindre pic. En
lecture seule, la sortie attendue est un constat : *service X, hôte Y, seuil actuel Z, mesure
observée sur 7 jours W, correctif proposé (seuil Z'), gain attendu (fin des faux positifs)* —
et rien d'autre. L'humain applique lui-même.

**Pourquoi commencer par là** : c'est la phase d'étalonnage. Quelques semaines de constats
suffisent à juger si les diagnostics de l'agent sont fiables avant de lui donner les clés.

## 6. Profil 2 — approbation manuelle (`hermes-ops`)

**Permissions** : profil 1 **+** Rule sets, Make changes, Audit log, Revert changes.
**Sans** Activate configuration.

**Boucle de travail** :

1. l'agent détecte l'anomalie de réglage ;
2. il écrit la règle (elle apparaît dans les *modifications en attente*) ;
3. il publie le constat : règle créée + identifiant, paramètres avant/après, données de mesure ;
4. **un humain active** — sans ce clic, rien ne change sur la supervision ;
5. en cas de désaccord, il supprime sa règle (ou l'humain la rejette) : aucun effet résiduel.

**Exemple** — un check HTTP dont le service tombe en alerte de façon intermittente parce que le
délai d'attente par défaut est trop court pour l'application surveillée. L'agent crée une règle
`[hermes] HTTP — délai 30 s` dans le dossier de test, la liste avec son identifiant, et
l'humain active. Si l'application est en réalité lente en permanence, le problème est applicatif :
l'humain supprime la règle, rien n'a été activé.

**Propriété intéressante** : l'agent peut préparer plusieurs correctifs d'avance, y compris
ceux qu'on refusera — le coût d'un refus est nul.

## 7. Profil 3 — full-automatique (`hermes-auto`)

**Permissions** : profil 2 **+** Activate configuration. (`Activate foreign changes` reste
inutile : l'agent n'active que ses propres modifications.)

**Boucle fermée** :

1. détecter : anomalie de réglage confirmée sur plusieurs jours de mesures ;
2. préparer : règle préfixée `[hermes]`, portée la plus étroite possible (hôte ou service) ;
3. activer : et **noter l'horodatage** ;
4. mesurer : sur une fenêtre définie (24 h par défaut) ;
5. conclure : si le service est stable → rapport court ; s'il se dégrade → **rollback
   automatique** (suppression de la règle + activation) et alerte.

**Exemple de rapport** :

> `[hermes]` hôte `srv-1`, service `Consommation prise` : WARNING récurrent depuis 6 jours,
> seuil 3 500 W → 4 200 W (mesures : médiane 3 900 W, pic 4 100 W). Règle activée à 14 h 02.
> Contrôle à 24 h : aucune alerte depuis l'activation.

**Ce qui rend ce profil acceptable** : le rollback est prévu dès la conception, l'activation est
horodatée, et le périmètre d'écriture reste un dossier de test tant que la confiance n'est pas
établie.

---

## 8. Écrire le correctif via l'API REST

Chemins relevés dans le code des endpoints (branche `master`) ✅ — le préfixe commun est
`https://<checkmk-host>/<site>/check_mk/api/1.0`.

| Action | Méthode et chemin |
|---|---|
| Lister les règles d'un rule set | `GET /domain-types/rule/collections/all?ruleset_name=<rule_set>` |
| Créer une règle | `POST /domain-types/rule/collections/all` |
| Lire / modifier / supprimer une règle | `GET` / `PUT` / `DELETE /objects/rule/<rule_id>` (modification sous `If-Match: <etag>`) |
| Déplacer une règle | `POST /objects/rule/<rule_id>/actions/move` |
| Modifications en attente | `GET /domain-types/activation_run/collections/pending_changes` |
| Activer | `POST /domain-types/activation_run/actions/activate-changes` |
| Suivre une activation | `GET /objects/activation_run/<id>` — `POST /objects/activation_run/<id>/actions/wait-for-completion` |
| Activations en cours | `GET /domain-types/activation_run/collections/running` |

Exemples :

```bash
BASE="https://<checkmk-host>/<site>/check_mk/api/1.0"

# 1. lire les règles d'un rule set (profil 1 suffit)
curl -s -H "Authorization: Bearer <utilisateur> <secret>" \
  "$BASE/domain-types/rule/collections/all?ruleset_name=<rule_set>" | jq

# 2. préparer un correctif (profil 2) — corps exact à confirmer ⚠️
curl -s -X POST -H "Authorization: Bearer <utilisateur> <secret>" \
  -H "Content-Type: application/json" \
  -d '{"ruleset": "<rule_set>", "folder": "/<dossier>",
       "value_raw": "<valeur brute>",
       "conditions": {"host_name": {"match_on": ["srv-1"], "operator": "one_of"}}}' \
  "$BASE/domain-types/rule/collections/all" | jq

# 3. activer (profil 3)
curl -s -X POST -H "Authorization: Bearer <utilisateur> <secret>" \
  -H "Content-Type: application/json" -d '{"sites": ["<site>"], "force_foreign_changes": false}' \
  "$BASE/domain-types/activation_run/actions/activate-changes" | jq
```

⚠️ La forme exacte du corps de création (`value_raw` et `conditions`) et l'usage de `If-Match`
en modification restent à valider sur un site réel : les valeurs brutes d'un rule set dépendent
du rule set visé. Le détail des modèles est dans `cmk/gui/openapi/api_endpoints/rule/models/`.

## 9. Garde-fous communs aux trois profils

- **Périmètre** : démarrer par un dossier de test, puis élargir. Le droit d'écriture global
  (`Write access to all hosts and folders`) n'est pas nécessaire au début.
- **Une règle par correctif**, préfixée `[hermes]`, avec la cause dans le nom (le rollback et la
  revue deviennent triviaux).
- **Jamais** `Add or modify executables` : les rule sets contenant des commandes brutes
  deviennent un vecteur d'exécution arbitraire sur les hôtes supervisés.
- **Jamais** `Notification configuration` : l'agent pourrait modifier ses propres alertes.
- **Jamais** les modules *Global settings*, *User management*, *Roles*, *Site management*,
  *Backup & restore*, *Password management* ; ni Event Console, BI, NagVis, snapshots.
- **Sans objet** : les sections *Topics* et *Dashboards* du rôle éditeur ne sont que de la
  visibilité (voir §4) ; les cases `… dashboards` de *General* ne servent qu'à créer des
  tableaux de bord personnels.
- **Journal d'audit** systématiquement activé, et relevé dans les rapports.
- **Activation horodatée** + fenêtre de mesure avant conclusion (profil 3).

## 10. Ce que ce rôle ne réglera pas

- **Les checks locaux maison** (scripts custom sans rule set paramétrable) : la correction est
  une modification de script, pas une règle — hors de portée de l'API.
- **L'activation est globale au site** : l'agent ne peut pas activer « seulement sa règle ».
  Si d'autres modifications sont en attente, elles partent avec (d'où l'intérêt de ne pas lui
  donner `Activate foreign changes`, qui l'autorise explicitement à embarquer celles des autres).
- **Les seuils qui ne sont pas des règles** : découverte de services, paramètres de check
  injectés autrement, configuration poussée par un outil externe.
- **Un problème applicatif déguisé en mauvais réglage** : remonter le seuil ne répare pas une
  application lente. C'est justement le cas que le profil 2 laisse trancher par un humain.

## 11. Vérification sur l'instance (16/09/2026)

Le compte d'automatisation **`hermes-read`** a été créé puis testé pour de vrai
contre l'instance (Checkmk **2.5.0p11**, community). Méthode :

1. la spec OpenAPI que l'instance sert elle-même (`/api/1.0/openapi-doc.yaml`)
   donne, pour chacun de ses 193 endpoints, **la liste exacte des permissions
   exigées** — c'est la source la plus fiable, elle décrit *cette* version ;
2. chaque endpoint a ensuite été appelé réellement : quand une permission manque,
   Checkmk répond 401 **en nommant la permission** — le libellé ci-dessous est
   donc rendu par l'instance, pas déduit ;
3. les sondes d'écriture visent des cibles **inexistantes** (règle sans jeu de
   règles, hôte dupliqué) : si la permission manque, l'appel échoue avant tout
   effet ; sinon il échoue sur la cible. Comptage avant/après identique
   (hôtes, dossiers, downtimes, commentaires inchangés) — **rien n'a été modifié**.

### Ce que le compte peut lire

- **Tout le parc, en supervision** : les hôtes, tous leurs services et leurs
  états, les downtimes, les commentaires, la console d'événements. Autrement dit
  `general.see_all` (« See all hosts and services ») est bien accordé — sans lui
  l'API ne renverrait que les hôtes des groupes de contacts de l'utilisateur.
- **L'inventaire du Setup** : la liste des hôtes et des dossiers
  (`wato.see_all_folders`, « Read access to all hosts and folders »).
- **Les règles, depuis l'ajout de `Rule sets`** : la liste des 42 jeux de règles
  qui en contiennent, les 21 règles du jeu `ignored_services`, et le contenu
  complet d'une règle lue par son identifiant (conditions, valeur, dossier).
- La console d'événements (`mkeventd.*` en lecture) — vide sur cette instance.
- La version de l'instance.

### Ce que le compte se voit refuser (libellé rendu par l'instance)

- `Make changes, perform actions` (`wato.edit`) → aucune écriture possible, y
  compris sur une règle que le compte lit par ailleurs (vérifié sur une vraie
  règle : `PUT`, `DELETE` et création répondent tous 401).
- `Activate configuration` (`wato.activate`) → rien ne peut être publié.
- `Audit log` (`wato.auditlog`), `User management` (`wato.users`),
  `Password management` (`wato.passwords`), `Site management` (`wato.sites`),
  `Host & service groups` (`wato.groups`), `Time periods` (`wato.timeperiods`),
  `Manage tags` (`wato.hosttags`), `Business Intelligence rules` (`wato.bi_rules`),
  `Edit personal notification settings` (`general.edit_notifications`).

### Verdict

**Rien en trop.** Aucun droit d'écriture, d'activation, d'administration des
utilisateurs, d'accès aux secrets, au journal d'audit ou aux sites — le compte ne
peut pas dégrader la supervision, même détourné.

**Un seul manquait pour l'usage visé** : `Rule sets` (`wato.rulesets`). Ajouté à
`hermes-read` le 16/09/2026, puis retesté : la lecture des règles fonctionne et
l'écriture reste refusée (§ « Confirmation en direct »). **Ni trop, ni pas assez.**

### Lire une règle sans jamais pouvoir la modifier

Permissions exigées par chaque opération, relevées dans la spec OpenAPI servie par
l'instance (`/check_mk/api/1.0/openapi-doc.yaml`). Quand plusieurs identifiants sont
listés, **tous** sont exigés :

| Opération | Permissions exigées |
|---|---|
| `GET /objects/rule/{rule_id}` | `wato.rulesets` |
| `GET /objects/ruleset/{ruleset_name}` | `wato.rulesets` |
| `GET /domain-types/rule/collections/all?ruleset_name=<nom>` | `wato.rulesets` |
| `POST /domain-types/rule/collections/all` (créer) | **`wato.edit`** + `wato.rulesets` + `wato.all_folders` |
| `PUT /objects/rule/{rule_id}` (modifier) | **`wato.edit`** + `wato.rulesets` + `wato.all_folders` |
| `DELETE /objects/rule/{rule_id}` (supprimer) | **`wato.edit`** + `wato.rulesets` + `wato.all_folders` |
| `POST /objects/rule/{rule_id}/actions/move/invoke` | **`wato.edit`** + `wato.rulesets` + `wato.all_folders` |

Autrement dit : **`wato.rulesets` seul ouvre la lecture ; toute écriture exige
`wato.edit` en plus.** La séparation n'est pas une convention de rédaction, elle est
appliquée par l'API.

Contrôle étendu à toute l'API (193 endpoints, 35 opérations exigeant une permission
de *portée* — `wato.all_folders` ou `wato.edit_all_passwords`) : les 7 lectures sont
des `GET`, et les 28 écritures réclament toutes une permission supplémentaire que le
profil ne possède pas (`wato.edit`, `wato.edit_hosts` ou `wato.rename_hosts`). Aucune
écriture n'est donc atteignable par la seule permission de portée.

**Le piège annoncé était un faux piège.** La spec OpenAPI de l'instance décrit
`GET /domain-types/rule/collections/all` (recherche tous jeux confondus) et
`GET /domain-types/ruleset/collections/all` (liste des jeux de règles) comme
exigeant aussi `wato.all_folders` et `wato.edit_all_passwords`. **Mesuré sur
l'instance : non.** Les deux endpoints répondent 200 avec `wato.rulesets` seul. La
description est générique à toute la famille d'endpoints d'écriture, pas
l'application réelle du contrôle. Conclusion pratique : `Rule sets` suffit à la
lecture, rien d'autre n'est à concéder.

**Un détail de chemin, lui, bien réel** : `GET /objects/ruleset/{nom}` répond 200
mais ne liste les règles que dans la portée du dossier courant — 0 règle en
pratique. Pour lire les règles d'un jeu : `GET
/domain-types/rule/collections/all?ruleset_name=<nom>`, puis
`GET /objects/rule/{id}` pour le contenu d'une règle.

`Activate configuration` (`wato.activate`) reste hors profil : **lire une règle ne
permet pas de l'appliquer**, et la configuration n'est jamais activée par un compte
de lecture.

### Confirmation en direct (16/09/2026, après ajout de `Rule sets`)

Permission ajoutée au compte, puis sondes rejouées. Réponses de l'instance :

| Ce que fait le compte | Réponse |
|---|---|
| lire les jeux de règles qui en contiennent (42) | 200 |
| lire les 21 règles du jeu `ignored_services` | 200 |
| lire une règle par son identifiant | 200 — conditions, valeur et dossier compris |
| modifier une règle (`PUT`) | **401 — `Make changes, perform actions`** |
| supprimer une règle (`DELETE`) | **401 — `Make changes, perform actions`** |
| créer une règle (`POST`) | **401 — `Make changes, perform actions`** |

**Lire oui, écrire non** — et ce n'est pas une convention de rédaction : c'est
l'instance qui refuse, sur la règle elle-même. Comptage avant/après identique
(29 hôtes, 7 dossiers, 48 downtimes, 48 commentaires, 42 jeux, 2 règles) : **rien
n'a été modifié**.

Reste fermé, vérifié au même moment : `Time periods`, `Edit personal notification
settings`, `User management`, `Host & service groups`, `Password management`,
`Audit log`, `Activate configuration`, `Site management`, `Manage tags`. Rien n'a
été ouvert « en passant » par l'ajout de `Rule sets`.

### Identifiants tels que l'instance les nomme

Table relevée dans la spec OpenAPI de l'instance (libellé ↔ identifiant) :

```
Activate configuration            wato.activate
Add & remove folders              wato.manage_folders
Add & remove hosts                wato.manage_hosts
Add comments                      action.addcomment
Acknowledge                       action.acknowledge
Agent pairing                     general.agent_pairing
Archive an event                  mkeventd.delete
Audit log                         wato.auditlog
Business Intelligence rules       wato.bi_rules
Change event state                mkeventd.changestate
Disabled services                 wato.service_discovery_to_ignored
Edit personal notification…       general.edit_notifications
Host & service groups             wato.groups
Host management                   wato.hosts
Make changes, perform actions     wato.edit
Manage background jobs            background_jobs.manage_jobs
Manage services                   wato.services
Manage tags                       wato.hosttags
Modify existing folders           wato.edit_folders
Modify existing hosts             wato.edit_hosts
Move existing hosts               wato.move_hosts
Move to monitored services        wato.service_discovery_to_monitored
Move to undecided services        wato.service_discovery_to_undecided
Password management               wato.passwords
Perform network parent scan       wato.parentscan
Read access to all hosts/folders  wato.see_all_folders
Read access to all modules        wato.seeall
Remove services                   wato.service_discovery_to_removed
Rename existing hosts             wato.rename_hosts
Rule sets                         wato.rulesets
Set/remove downtimes              action.downtimes
Site management                   wato.sites
Time periods                      wato.timeperiods
Update an event                   mkeventd.update
User management                   wato.users
Write access to all hosts/folders wato.all_folders
Write access to all passwords     wato.edit_all_passwords
```

Les sondes sont rejouables : `probe_api4.py` (lecture + écriture sans effet) et
`spec_perms.py` (matrice endpoint → permissions, extraite de la spec).

## 12. Ce qui est vérifié, ce qui reste à tester

- ✅ Permissions et séparation écrire/activer (`cmk/gui/wato/_permissions.py`,
  `cmk/gui/openapi/api_endpoints/rule/_utils.py`,
  `cmk/gui/openapi/endpoints/activate_changes/__init__.py`).
- ✅ Chemins d'API des endpoints `rule` et `activation_run`.
- ✅ Séparation lecture/écriture sur les règles **mesurée en direct** : `Rule sets`
  accordé → lecture des règles et des jeux de règles ; toute écriture (créer,
  modifier, supprimer) refusée avec `Make changes, perform actions`.
- ✅ Section *Topics* du rôle éditeur : permissions dynamiques des pages
  (`cmk/gui/pagetypes/_core.py`, `declare_permission_section` / `declare_permission`) —
  visibilité des thèmes, sans droit d'écriture.
- ⚠️ Création/modification de règle par un compte *autorisé* à écrire : corps exact du
  message, gestion des `etag`, comportement sur un dossier restreint — impossible à
  tester avec `hermes-read`, qui n'a pas le droit d'écrire (c'est le but).
- ⚠️ Comportement du rollback automatique et fenêtre de mesure pertinente.
- ⚠️ Le compte d'automatisation `hermes-read` existe et a été testé (§11). Les deux
  autres profils (`hermes-ops`, `hermes-auto`) ne sont pas créés : rien n'est encore
  appliqué en écriture sur la production.

## 13. Journal des mises à jour

- **16/09/2026** — §11 : retest après ajout de `Rule sets` à `hermes-read` — lecture des
  règles confirmée (42 jeux, 21 règles d'un jeu, une règle par identifiant), écriture
  toujours refusée (401 `Make changes, perform actions`) ; le « piège » `Write access to
  all hosts and folders` annoncé par la spec s'avère non appliqué.

- **16/09/2026** — §11 : résultats des tests réels du compte `hermes-read` sur l'instance
  (matrice endpoint → permissions extraite de la spec OpenAPI 2.5, permissions
  manquantes nommées par l'instance, sondes d'écriture sans effet, verdict « rien en
  trop, il manque *Rule sets* »).

- **16/09/2026** — §4 : la liste des cases à cocher, rôle par rôle, et la clarification de la
  section *Topics* (visibilité seule). §3 : identifiant de *See all host and services*
  corrigé en `general.see_all`.
- **16/09/2026** — Première version : les trois profils, les permissions vérifiées dans le code
  source, les chemins d'API, les garde-fous. Reste à trancher : démarrer en profil 2 ou en
  profil 3, et portée (tous les hôtes ou dossier de test d'abord).
