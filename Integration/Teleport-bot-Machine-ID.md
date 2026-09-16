# Intégration d'Hermes à Teleport : bot Machine ID pour un accès SSH au parc

Ce guide documente une intégration **testée et fonctionnelle** : un agent Hermes
(c'est-à-dire l'IA qui exécute des tâches sysadmin) se connecte en SSH à tout un
parc de machines **sans aucune clé SSH, sans mot de passe et sans secret stocké
sur les machines cibles**. L'accès repose sur un *bot* Teleport (Machine ID /
`tbot`) qui obtient des certificats SSH de courte durée, renouvelés automatiquement.

Il complète `[IN PROGRESS] Teleport.md` (ébauche) en décrivant la version qui a
réellement tourné, avec les pièges rencontrés sur le terrain.

Version testée : **Teleport 15.5.4**, agent Hermes sur un hôte Linux, bastion
Teleport sur une VM Debian.

> [!note]
> Aucune valeur d'infrastructure réelle n'apparaît dans ce document (pas de nom
> d'hôte, d'adresse, de domaine, de token ni d'empreinte de certificat) : tout est
> remplacé par des valeurs d'exemple. Adaptez-les à votre environnement.

---

## 1. Objectif et principe

Objectif concret visé ici : **root sur tout le parc, sauf le bastion lui-même**.

Le principe, en trois points :

1. **Un bot, pas un utilisateur.** `tbot` rejoint le cluster Teleport avec un
   *join token* (jetable et à durée limitée), puis se présente avec une identité
   qui lui est propre.
2. **Des certificats courts et renouvelés.** Le bot demande une identité de 1 h,
   renouvelée toutes les 20 min, et l'écrit sur disque dans un fichier lisible
   uniquement par l'utilisateur qui exécute l'agent (`0600`). C'est ce fichier
   que `tsh` utilise pour se connecter.
3. **Le périmètre est décidé par un rôle Teleport.** Le bot ne « possède » aucun
   droit : il emprunte un rôle. Modifier ou supprimer le rôle coupe l'accès
   immédiatement, sans rien changer sur les machines cibles ni sur l'hôte Hermes.

Avantages en pratique : rien à installer sur les machines cibles, aucune clé à
distribuer, aucune clé publique à mettre dans un `authorized_keys`, et une trace
d'audit côté Teleport (session recording) pour chaque commande exécutée.

## 2. Prérequis

- Un cluster Teleport **15.x ou plus** (testé sur 15.5.4) avec un accès admin
  `tctl` sur le bastion.
- Sur l'hôte Hermes : les binaires `tbot` + `tsh` en version **identique** à
  celle du cluster.
- Un accès sortant TCP 443 de l'hôte Hermes vers le proxy Teleport. **Aucun port
  entrant** n'est nécessaire sur l'hôte Hermes.
- Le nom des nœuds du parc est déjà connu du cluster Teleport (les nœuds ont un
  agent Teleport ou SSH Service actif).

## 3. Côté bastion (administration, en `root`)

### 3.1 Écrire le rôle du bot

C'est **l'étape la plus importante** et celle qui m'a coûté le plus de temps :
la façon dont on désigne les machines autorisées change tout (voir la section 9,
piège n°1).

```yaml
kind: role
version: v6            # v6 sur Teleport 15.x ; v7 sur les versions 18.x
metadata:
  name: hermes-agent-role
spec:
  allow:
    logins: ['root']            # le login UNIX utilisé côté cible
    node_labels:
      hostname: ['*']           # tout nœud PORTANT un label "hostname"
  deny:
    node_labels:
      hostname: ['bastion']     # ceinture et bretelles
  max_session_ttl: 8h
```

**Pourquoi `hostname: ['*']` et non `'*': '*'` ?** Parce que chaque nœud sain
enregistré dans le cluster porte automatiquement un label `hostname`, alors que
le nœud correspondant au bastion lui-même peut s'enregistrer **sans aucun
label**. Un `allow.node_labels` exprimé en wildcard total (`'*': '*'`) matche
aussi un nœud sans labels : le bot obtient alors root sur le bastion, et la
règle `deny` visant `hostname: bastion` ne s'applique jamais (le label n'existe
pas sur ce nœud) — échec silencieux, aucune erreur, juste un accès trop large.

En scopant l'`allow` sur `hostname: ['*']`, un nœud sans labels est exclu par
l'autorisation elle-même, sans dépendre d'un label absent. La règle `deny`
devient un filet de sécurité.

### 3.2 Créer le rôle sur le cluster

```bash
tctl create -f hermes-role.yaml
tctl get role/hermes-agent-role --format=yaml     # vérifier ce qui a été pris en compte
```

Vérifiez la sortie de `tctl get` : c'est elle qui fait foi. Si vous ne voyez pas
votre `node_labels`, le YAML n'a pas été interprété comme vous le croyez (voir
piège n°4).

### 3.3 Créer le bot et récupérer son token

```bash
tctl bots add hermes-agent --roles=hermes-agent-role --ttl=30m
```

La commande affiche un token d'invitation. Règles d'hygiène :

- le token est un secret **à usage unique et à durée limitée** : ne le committez
  jamais, ne le collez pas dans un chat, ne le laissez pas dans un historique de
  shell ;
- déposez-le sur l'hôte Hermes dans un fichier `0600` appartenant à l'utilisateur
  qui fait tourner le bot, et référencez **le chemin** du fichier dans la
  configuration (jamais la valeur, voir 4.3).

### 3.4 Nettoyage

Si vous avez ouvert un accès SSH temporaire (clé « bootstrap ») pour préparer le
bastion, **retirez-la** une fois le bot créé : l'intérêt du montage est justement
de ne plus dépendre d'une clé. Vérifiez ensuite depuis l'hôte Hermes que cet
accès direct est bien refusé (`Permission denied`).

## 4. Côté hôte Hermes (l'agent)

### 4.1 Installer les binaires

```bash
curl -O https://cdn.teleport.dev/teleport-v15.5.4-linux-amd64-bin.tar.gz
tar -xzf teleport-v15.5.4-linux-amd64-bin.tar.gz
cd teleport && ./install
```

### 4.2 Créer les répertoires

```bash
mkdir -p /var/lib/teleport/bot /opt/machine-id
chmod 700 /var/lib/teleport/bot
```

- `/var/lib/teleport/bot` : données internes du bot (état, certificat
  renouvelable). À protéger : qui lit ce dossier peut se faire passer pour le bot.
- `/opt/machine-id` : destination des certificats courts, consommés par `tsh`.

### 4.3 Écrire `/etc/tbot.yaml`

```yaml
version: v2
proxy_server: teleport.example.lan:443
onboarding:
  join_method: token
  token: /etc/teleport/bot-token          # chemin d'un fichier 0600, pas la valeur
  ca_pins:
    - sha256:REMPLACER_PAR_EMPREINTE_CA
storage:
  type: directory
  path: /var/lib/teleport/bot
certificate_ttl: 1h
renewal_interval: 20m
services:
  - type: identity
    destination:
      type: directory
      path: /opt/machine-id
```

Points d'attention :

- `ca_pins` : l'empreinte de l'autorité de certification du cluster, affichée par
  `tctl status` sur le bastion. C'est la méthode propre pour un cluster dont le
  certificat n'est pas dans les magasins système. À défaut, on peut installer le
  certificat du cluster sur l'hôte Hermes, ou (dernier recours, à éviter)
  `insecure: true`.
- `certificate_ttl` doit être supérieur à `renewal_interval`, sinon le bot
  renouvelle des certificats déjà expirés.
- Sur Teleport 15.x, le seul type de sortie disponible est `identity` : le
  certificat produit est consommé par `tsh` (voir section 5). Le type de sortie
  `ssh`/`openssh`, qui produit une configuration OpenSSH classique, n'arrive
  qu'avec les versions 16+.

### 4.4 Déposer le token, puis valider avant de passer en service

```bash
umask 077
install -m 600 /dev/stdin /etc/teleport/bot-token   # collez le token, puis Ctrl-D
shred -u <copie-temporaire-du-token>

tbot start -c /etc/tbot.yaml --oneshot              # test : doit sortir sans erreur
```

Un `--oneshot` réussi signifie que le bot a joint le cluster et écrit son
identité. Ne passez à l'étape suivante qu'après ce test.

### 4.5 Installer et démarrer le service

```bash
tbot install systemd -c /etc/tbot.yaml --write --name tbot --user=root --group=root
systemctl enable --now tbot
systemctl is-active tbot && systemctl is-enabled tbot
```

L'unité générée relance le bot en cas d'arrêt (`Restart=always`) et désactive la
télémétrie anonyme de Teleport (`TELEPORT_ANONYMOUS_TELEMETRY=0`).

> [!warning]
> Faire tourner le bot en `root` avec un rôle qui donne `root` sur le parc est le
> montage le plus simple, mais ce n'est pas le plus sûr. En production, préférez
> un utilisateur dédié côté cible (par exemple `hermes` ou `ansible`), un rôle
> par usage, et un `max_session_ttl` court.

## 5. Utiliser l'accès depuis Hermes

```bash
# Lister les nœuds accessibles avec le rôle du bot
tsh -i /opt/machine-id/identity --proxy=teleport.example.lan:443 ls

# Exécuter une commande
tsh -i /opt/machine-id/identity --proxy=teleport.example.lan:443 ssh root@node1 'hostname; id -un'

# Copier un fichier
tsh -i /opt/machine-id/identity --proxy=teleport.example.lan:443 scp fichier root@node1:/tmp/
```

Pour éviter de répéter les options, définissez un alias ou une variable :

```bash
TSH="tsh -i /opt/machine-id/identity --proxy=teleport.example.lan:443"
$TSH ls
```

**Ansible et SSH natif.** Sur Teleport 15.x, `tbot` ne sait pas produire de
certificats OpenSSH « classiques » : l'identité passe par `tsh`, donc un
`ssh` natif ou Ansible ne fonctionne pas directement. Deux options :

- passer à Teleport 16+ et utiliser la sortie `ssh` de `tbot`, qui écrit
  certificats et configuration OpenSSH exploitables par `ssh`/Ansible ;
- rester en 15.x et générer un certificat OpenSSH côté administration avec
  `tctl auth sign --format=openssh ...` (la commande est disponible en 15.x).

## 6. Vérifier que le périmètre est bien celui voulu

Un accès qui « marche » ne prouve pas qu'il est correctement limité : testez les
**deux** côtés.

```bash
# 1) Le bot est bien identifié et a le bon rôle
$TSH status

# 2) Le parc est visible
$TSH ls

# 3) Test positif : une machine du parc répond
$TSH ssh root@node1 'echo OK'
#    -> OK / <nom du nœud> / root

# 4) Test négatif : le bastion est refusé
$TSH ssh root@bastion 'echo INTERDIT'
#    -> ERROR: access denied to root connecting to bastion:0

# 5) Le service tourne et renouvelle
systemctl is-active tbot; systemctl is-enabled tbot
ls -l /opt/machine-id/identity      # horodatage < certificate_ttl
```

Si le test 4 passe alors qu'il devrait échouer, le rôle est trop permissif :
retournez en section 3.1.

## 7. Ajouter une machine au parc

Rien à faire : le label `hostname` est posé automatiquement à l'enregistrement du
nœud, donc le nouveau serveur entre dans le périmètre du rôle tout seul. Contrôle
utile : le nœud doit apparaître dans `$TSH ls` ; sinon la connexion échouera
proprement en `access denied`.

## 8. Couper l'accès

```bash
tctl bots rm hermes-agent            # suppression du bot
# ou, sans supprimer le bot :
tctl bots lock hermes-agent          # verrouille l'identité (recommandé en cas de doute)
```

L'accès tombe à l'expiration du certificat en cours (au plus `certificate_ttl`).
Aucune action n'est requise sur les machines cibles.

## 9. Pièges rencontrés (retour d'expérience)

1. **`'*': '*'` autorise aussi un nœud sans labels.** Le wildcard total matche
   un nœud qui ne porte aucun label — et c'est justement le cas du nœud du
   bastion. Résultat : le bot obtient root sur le bastion, et la règle `deny`
   visant ce nœud ne se déclenche jamais car le label visé n'existe pas.
   → Scoper l'`allow` sur un label réellement présent (`hostname: ['*']`).
2. **Vérifier les labels réels avant d'écrire une règle.** `tsh ls -v` montre un
   label `hostname=<nom>` pour la plupart des nœuds… et rien du tout pour le
   nœud du bastion. Une règle `deny` ne peut pas fonctionner sur un label absent.
3. **`tbot` 15.x ne connaît pas le type de sortie `openssh`.** Une config qui en
   contient une fait échouer le démarrage avec
   `failed parsing config file: unrecognized service type (openssh)`.
   → `identity` + `tsh` en 15.x, sortie `ssh` à partir de la 16.
4. **Un YAML collé à la main peut perdre ses quotes.** Un `'*'` recopié via un
   chat ou recollé avec une indentation différente peut être interprété
   différemment (voire ignoré) par `tctl`, sans erreur. Méthode fiable pour
   livrer un rôle : transférer le fichier encodé et vérifier son empreinte avant
   application.

   ```bash
   # sur la machine qui rédige le rôle
   base64 -w0 hermes-role.yaml     # à transmettre
   sha256sum hermes-role.yaml      # à comparer
   # sur le bastion
   echo '<blob_base64>' | base64 -d > /root/hermes-role.yaml
   sha256sum /root/hermes-role.yaml       # doit être identique
   tctl create -f /root/hermes-role.yaml --force
   ```
5. **`tsh` met en cache l'identité dans `~/.tsh`.** Après un changement de rôle
   ou de certificat, l'ancien contexte peut continuer d'être utilisé et fausser
   les tests. → `rm -rf ~/.tsh` puis rejouer la commande.
6. **Ne jamais mettre le token en clair dans `tbot.yaml`.** Un fichier `0600`
   référencé par chemin, et le token retiré/détruit après le premier join. Un
   token de join qui traîne dans un dépôt Git public doit être considéré comme
   compromis et supprimé (`tctl tokens rm <token>`).
7. **Retirer la clé de bootstrap** sur le bastion après la création du bot, et
   vérifier que l'accès direct par clé est bien refusé.
8. **Le bastion n'a pas à être accessible par le bot.** Si vous y accédez par
   `tsh`, utilisez votre propre identité d'administrateur, pas celle du bot.

## 10. Limites de ce guide

Ce guide ne couvre pas : les accès humains (SSO, `tsh login` interactif), les
applications web et bases de données (autres types de sortie de `tbot`), la
rotation ou le verrouillage avancé des bots, ni le durcissement du bastion
lui-même. Voir la documentation officielle de Teleport, section *Machine &
Workload Identity*.
