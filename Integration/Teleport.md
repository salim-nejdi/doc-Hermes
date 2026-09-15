# Guide d'intégration d'Hermes à la solution de bastion Teleport
Ce guide porte sur la configuration de Teleport et d'Hermes pour éviter d'utiliser des clés SSH ou des mots de passe pour se connecter à des machines afin d'y automatiser des tâches de sysadmin.

# Contexte technique
La solution de bastion Teleport est ici installée sur une machine virtuelle Debian 13.

# Mise en place
## Côté serveur Teleport
Connectez-vous au shell de votre bastion Teleport.

1. Créez le fichier hermes-role.yaml qui servira à définir les permissions (le rôle) dédiées à Hermes :
```
cat <<EOF > hermes-role.yaml
kind: role
version: v7
metadata:
  name: hermes-agent-role
spec:
  allow:
    logins: ['ansible', 'hermes', 'debian']
    node_labels:
      '*': '*'
    app_labels:
      '*': '*'
    rules:
      - resources: [session]
        verbs: [read, list]
        where: contains(session.participants, user.metadata.name)
  deny:
    logins: ['root']
  options:
    max_session_ttl: 1h0m0s
EOF
```

2. Appliquez le YAML pour créer le rôle sur le cluster :
```
tctl create -f hermes-role.yaml
```

3. Créez la ressource de type bot pour Hermes. Conservez précieusement le token généré par cette commande
```
tctl bots add hermes-bot --roles=hermes-agent-role
```

## Côté serveur Hermes

1. Commencez par télécharger l'archive Teleport (adaptez la version si nécessaire) :
```
curl -O https://cdn.teleport.dev/teleport-v18.11.0-linux-amd64-bin.tar.gz
```

2. Procédez à l'extraction de l'archive :
```
tar -xzf teleport-v18.11.0-linux-amd64-bin.tar.gz
```

3. Déplacez-vous dans l'archive décompressée et installez les binaires Teleport :
```
cd teleport
sudo ./install
```

4. Créez le dossier où tbot va stocker ses données internes et attribuez-le à root :
```
mkdir -p /var/lib/teleport/bot
chown -R root:root /var/lib/teleport/bot
```

5. Créez les sous-dossiers où les certificats utilisables seront générés pour chaque service :
```
mkdir -p /opt/machine-id/{ssh,checkmk,proxmox}
```

6. Créez le fichier de configuration /etc/tbot.yaml. Pensez à remplacer la valeur du token par celui généré à l'étape 4 du serveur Teleport :
```
version: v2
auth_server: "teleport.tadaron:443"
onboarding:
  join_method: "token"
  token: "d577ae2968208205c765cea54db17c75"
storage:
  type: directory
  path: /var/lib/teleport/bot
outputs:
  # 1. Configuration pour l'accès SSH
  - type: ssh_client
    destination:
      type: directory
      path: /opt/machine-id/ssh
      
  # 2. Configuration pour l'accès API Checkmk
  - type: application
    app_name: checkmk
    destination:
      type: directory
      path: /opt/machine-id/checkmk
      
  # 3. Configuration pour l'accès API Proxmox
  - type: application
    app_name: proxmox
    destination:
      type: directory
      path: /opt/machine-id/proxmox
```

7. Initialisez les dossiers de destination. Cette étape permet d'autoriser l'utilisateur Linux qui exécute l'agent Hermes (ici nommé UTILISATEUR_HERMES) à lire les certificats.  
Remplacez cette valeur par le nom d'utilisateur réel :
```
tbot init -c /etc/tbot.yaml --bot-user=root --reader-user=UTILISATEUR_HERMES --init-dir=/opt/machine-id/ssh
```

8. Démarrez le bot pour générer les certificats. Le processus va tourner en premier plan, vérifiez qu'aucune erreur ne s'affiche :
```
tbot start -c /etc/tbot.yaml
```
