# Superviser les conteneurs Docker d'un parc, et détecter les mises à jour d'images

## Objectif

Deux besoins distincts, souvent confondus :

1. **savoir** — voir dans la supervision qu'un conteneur est arrêté, qu'il consomme trop,
   ou qu'une image a une version plus récente ;
2. **agir** — télécharger la nouvelle image et recréer le conteneur.

Ils se traitent avec deux outils séparés : l'**agent Checkmk** (savoir) et
**WUD / What's-up-docker** (savoir, et éventuellement agir). Rien n'oblige à faire les deux.

Versions testées, sur **Debian 13 (trixie)** : Checkmk **2.5.0p11 community**,
WUD **9.0.2**, `python3-docker` **7.1.0-2** (paquet Debian), Docker CE 29.x.

## 1. Ce que Checkmk voit — et ne voit pas

L'agent Checkmk d'une machine Linux remonte les paquets, les systèmes de fichiers, le
réseau, les comptes : **aucun conteneur**. Un serveur qui héberge dix conteneurs est
« OK » même si les dix sont morts.

Pour les voir, il faut deux choses, dans cet ordre :

- côté **machine supervisée** : le plugin d'agent `mk_docker.py` ;
- côté **supervision** : une *découverte de services* (chaque conteneur devient un
  service), puis une *activation* de la configuration.

Tant que la découverte n'est pas faite, le plugin peut remonter les conteneurs
correctement : il ne se passe rien de visible. C'est le piège numéro un.

## 2. Étape 1 — la dépendance Python, d'abord

`mk_docker.py` est un script Python qui parle au socket Docker via la bibliothèque
`docker`. **Sans elle, il ne plante pas** : il écrit un message d'erreur *à l'intérieur*
de sa section `docker_node_info`, là où personne ne le lit.

```bash
apt-get install -y --no-install-recommends python3-docker
python3 -c 'import docker; print(docker.__version__)'      # attendu : 7.1.0
```

Ce que l'on obtient sans la dépendance (test négatif, à vérifier avant d'aller plus loin) :

```bash
python3 /usr/lib/check_mk_agent/plugins/mk_docker.py | grep -o '{.*}'
# {"Critical": "Error: mk_docker requires the docker library.
#              Please install it on the monitored system (pip3 install docker)."}
```

## 3. Étape 2 — poser le plugin d'agent

Le fichier est livré avec le serveur de supervision, dans
`/omd/sites/<site>/share/check_mk/agents/plugins/mk_docker.py` (il est compilé, .pyc ou
non, selon l'installation — le prendre sur le serveur garantit la version qui correspond
à la supervision).

```bash
# 1. sur le serveur de supervision : le transport en base64 évite tout souci d'encodage
base64 -w0 /omd/sites/<site>/share/check_mk/agents/plugins/mk_docker.py > /tmp/mk_docker.b64

# 2. sur la machine supervisée
base64 -d /tmp/mk_docker.b64 > /usr/lib/check_mk_agent/plugins/mk_docker.py
chmod 0755 /usr/lib/check_mk_agent/plugins/mk_docker.py
```

Vérification (les conteneurs doivent apparaître, pas seulement la machine) :

```bash
python3 /usr/lib/check_mk_agent/plugins/mk_docker.py | grep '^<<<' | sed 's/:.*//' | sort -u
# docker_container_status, docker_container_cpu, docker_container_mem,
# docker_container_network, docker_container_labels, docker_container_diskstat,
# docker_container_node_name, docker_node_info, docker_node_images,
# docker_node_disk_usage, docker_node_network
```

Lecture utile : le plugin publie **une section par conteneur**, précédée d'un en-tête
*piggyback* formé des 12 premiers caractères de l'identifiant du conteneur
(`<<<<0123456789ab>>>>`). C'est ce qui permet à Checkmk d'attribuer chaque métrique au
bon conteneur.

Deux lignes de bruit sont normales : le plugin essaie d'exécuter `check_mk_agent`
**dans** chaque conteneur (pour récupérer une éventuelle supervision interne) et écrit
sur la sortie d'erreur, pour chaque conteneur qui n'en a pas :

```
running check_mk_agent in container 0123456789ab failed: OCI runtime exec failed:
exec failed: unable to start container process: exec: "check_mk_agent": executable file not found in $PATH
```

## 4. Étape 3 — découvrir les services, puis activer

Côté WebUI : `Setup ▸ Hosts`, cocher la machine, puis l'icône *Run service discovery*
(ou `Setup ▸ Hosts ▸ <hôte> ▸ Services ▸ Run service discovery`), accepter les services
proposés, puis **Activate changes**. Chaque conteneur devient un service
(`Container <nom>`, plus les métriques CPU/mémoire/réseau/disque).

Les plugins de contrôle correspondants sont fournis avec Checkmk
(`cmk/plugins/collection/agent_based/docker_*` en 2.5) : rien à installer côté serveur.

**Côté API, ces deux opérations ne sont pas accessibles à un compte en lecture seule.**
Mesuré sur l'instance : même la *lecture* du tableau de découverte
(`GET /objects/service_discovery/<hôte>`) répond 401 en nommant la permission
manquante — `Make changes, perform actions` (`wato.edit`). La découverte et
l'activation sont donc un geste d'administrateur, pas une étape automatisable avec un
compte de lecture.

## 5. Étape 4 — WUD : savoir qu'une image a une version plus récente

WUD surveille les conteneurs de la machine et compare les images locales à celles du
registre. Exemple de `docker-compose.yml` qui fonctionne (le nom de service n'a pas
d'importance, le conteneur non plus) :

```yaml
services:
  whatsupdocker:
    image: getwud/wud
    container_name: wud
    restart: unless-stopped
    stop_grace_period: 30s
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - wud_store:/store            # sinon l'historique est perdu à chaque recréation
    ports:
      - 3030:3000
    environment:
      - WUD_WATCHER_LOCAL_CRON=0 1 * * *
      - WUD_WATCHER_LOCAL_WATCHATSTART=true
      - WUD_REGISTRY_HUB_PUBLIC_WATCHDIGEST=true
      - WUD_AUTH_BASIC_ADMIN_USER=wud
      - WUD_AUTH_BASIC_ADMIN_HASH=REMPLACER_PAR_LE_HASH
      - WUD_TRIGGER_DISCORD_MONDISCORD_URL=REMPLACER_PAR_L_URL_DU_WEBHOOK
      - WUD_TRIGGER_DISCORD_MONDISCORD_MODE=simple

volumes:
  wud_store:
```

Le hash du mot de passe de l'interface se fabrique avec l'image elle-même :

```bash
docker run --rm getwud/wud node -e \
  "console.log(require('bcryptjs').hashSync(process.argv[1], 8))" 'mot-de-passe'
```

### Pourquoi ces trois lignes ne sont pas facultatives

| réglage | sans lui |
| --- | --- |
| `restart: unless-stopped` | le conteneur ne revient **jamais** tout seul (voir les pièges) |
| `stop_grace_period: 30s` | Docker le tue au bout de 10 s, à chaque redémarrage du démon |
| `..._PUBLIC_WATCHDIGEST=true` | avec des images taguées `latest`, WUD ne détecte **rien** |

`WUD_WATCHER_LOCAL_WATCHBYDIGEST` a existé mais est **déprécié** : dans WUD 9.x, le suivi
par digest se règle au niveau du **registre** (`WUD_REGISTRY_<REGISTRE>_<NOM>_WATCHDIGEST`),
pas du watcher. Écrire la clé dépréciée ne provoque aucune erreur : le réglage est
simplement ignoré.

Le nom exact des clés se lit dans le journal au démarrage — c'est la seule source
fiable, elle change d'une version à l'autre :

```bash
docker logs wud | grep 'Register with configuration'
# [registry.hub.public] Register with configuration {"watchdigest":true,...}
# [watcher.docker.local] Register with configuration {"cron":"0 1 * * *","watchatstart":"true"}
```

### Vérifier pour de vrai (avant/après)

```bash
docker logs wud --since 5m | grep -E 'Cron (started|finished)|available updates'
```

- **avant correction** : `not a semver and digest watching is disabled, so wud won't
  report any update` puis `2 containers watched, 0 available updates` — tous les jours,
  pour toujours ;
- **après** : `Watching digest for image <image>` puis
  `Cron finished (N containers watched, 0 errors, M available updates)`.

Test à blanc, hors production : lancer un second conteneur WUD avec la même
configuration, en cron déclenché au démarrage, et lire son journal. C'est ce qui permet
de valider une clé de configuration sans toucher au conteneur en service.

## 6. Pièges

- **Docker tue WUD, et WUD ne revient pas.** Une mise à jour du paquet `docker-ce`
  redémarre le démon, qui envoie `SIGTERM` à tous les conteneurs. Ceux qui répondent en
  moins de 10 s reviennent seuls ; WUD (Node.js) ne répond pas dans ce délai, Docker le
  tue (`Container failed to exit within 10s of signal 15 - using the force`) → sortie
  **137**, et sans `restart:` il reste mort **indéfiniment**. Constat sur trois machines
  différentes de la même flotte : mortes le même jour, personne ne l'a vu. La cause
  n'était pas WUD, c'était le démon.
- **Un conteneur mort ne remonte nulle part** : sans le plugin `mk_docker`, aucune
  supervision ne voit la différence entre un serveur qui tourne et un serveur dont tous
  les conteneurs sont arrêtés.
- **Le trigger Docker met à jour tout seul.** Déclarer une clé
  `WUD_TRIGGER_DOCKER_LOCAL_*`, c'est enregistrer le *trigger* Docker : dès qu'une image
  plus récente est trouvée, WUD **arrête, supprime et recrée le conteneur** avec la
  nouvelle image — et supprime l'ancienne (`{"prune":"true"}`). Observé en production sur
  un conteneur tiers, au premier passage de veille. Pour être *notifié* sans être
  *modifié*, ne pas déclarer ces clés (garder seulement `WUD_TRIGGER_DISCORD_...`).
- **Un nom de clé lu dans le code n'est pas le nom à employer.** Le même logiciel change
  d'orthographe selon la version (8.x lisait `..._WATCHDIGEST`, 9.x
  `..._WATCHDIGESTDEFAULT...`) ; se fier au journal de démarrage, pas au code trouvé en
  ligne.
- **`latest` n'est pas une version.** Sur des images taguées `latest`, le suivi
  « semver » ne peut rien comparer : le suivi par digest est le seul qui détecte quelque
  chose.
- **Une mise à jour de WUD vers 9.x change l'authentification** : les clés
  `WUD_AUTH_BASIC_ADMIN_USER/HASH` restent lues, mais l'interface répond désormais 200
  sans identifiants et protège l'API. Après migration, `docker logs` affiche
  `[store-migrate-loki] Migrated app version 8.x.x` : l'historique est repris, pas perdu.

## 7. Limites

- Le plugin d'agent **ne surveille pas ce qui n'est pas du Docker** : pas de Podman,
  pas de conteneurs Kubernetes (autre plugin).
- Sans découverte + activation côté supervision, l'installation du plugin est sans effet
  visible — et ces deux gestes demandent un compte administrateur (§4).
- WUD ne met pas à jour un conteneur créé par `docker compose` de la façon dont compose
  le ferait : il le recrée à l'identique (mêmes volumes, mêmes variables) mais les
  métadonnées de compose deviennent obsolètes. Le prochain `docker compose up -d` du
  projet recréera le conteneur pour se réaligner — sans dégât, mais c'est à savoir.
- Le suivi par digest interroge le registre : sur beaucoup d'images, garder le cron
  quotidien et non « toutes les cinq minutes » (avertissement
  `may result in throttled requests` dans le journal).
