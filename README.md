# Installer GitLab dans Docker avec Docker Compose

Ce guide accompagne la vidéo consacrée au déploiement d'une instance GitLab Community Edition auto-hébergée avec Docker Compose.

L'objectif est de mettre en place une installation structurée dans `/srv/gitlab`, accessible par HTTP sur le port `8080` et utilisable en SSH pour Git sur le port `2222`.

Cette configuration convient à un VPS Linux ou à un homelab. Elle constitue une première installation : HTTPS, le reverse proxy NGINX et les runners GitLab seront abordés dans d'autres vidéos.

> Les commandes ci-dessous sont prévues pour un serveur Ubuntu ou Debian sur lequel Docker Engine et le plugin Docker Compose sont déjà installés.
> Exécutez-les avec un utilisateur qui possède les droits `sudo` et peut utiliser la commande `docker`.

> Dans ce guide, remplacez `git.example.com` par le nom d'hôte de votre instance GitLab. Pour un homelab local, vous pouvez par exemple utiliser `git.homelab.local`, à condition que ce nom soit résolu depuis les postes qui accèdent à GitLab.

## Ressources liées

Cette page sert de support à la vidéo YouTube associée.

- YouTube : [Installer Gitlab dans Docker avec Docker Compose](https://youtu.be/Ik7sS_qr6WM)
- LinkedIn : [me suivre sur LinkedIn](https://www.linkedin.com/in/dominique-korzeczek-695033188)

## Prérequis

- Un VPS ou un homelab sous Linux.
- Un accès SSH avec un compte disposant de `sudo`.
- Docker Engine et Docker Compose installés.
- Les ports TCP `8080` et `2222` disponibles sur le serveur et accessibles depuis les clients qui utiliseront GitLab.
- Un nom d'hôte qui résout vers le serveur, si vous utilisez un nom comme `git.example.com`.

Vérifiez Docker et Docker Compose avec :

```bash
docker --version
docker compose version
```

> Cette première installation utilise HTTP sur le port `8080`. Elle ne doit pas être exposée telle quelle sur Internet pour un usage durable : ajoutez HTTPS avec un reverse proxy avant de l'utiliser en production.

## 1. Créer l'arborescence et le groupe d'administration

Créez les dossiers nécessaires à l'installation :

```bash
sudo mkdir -p /srv/gitlab/{compose,config,data,logs}
```

Créez le groupe `admin-gitlab` :

```bash
sudo groupadd admin-gitlab
```

Si ce groupe existe déjà, il n'est pas nécessaire de le recréer.

Ajoutez l'utilisateur actuellement connecté à ce groupe, puis activez-le dans le shell courant :

```bash
sudo usermod -aG admin-gitlab "$USER"
newgrp admin-gitlab
```

Attribuez l'arborescence à `root` et au groupe `admin-gitlab`, puis accordez les droits nécessaires au groupe :

```bash
sudo chown -R root:admin-gitlab /srv/gitlab
sudo chmod -R g+rwX /srv/gitlab
sudo find /srv/gitlab -type d -exec chmod g+s {} +
```

Le bit `setgid` appliqué aux dossiers permet aux nouveaux fichiers et sous-dossiers d'hériter du groupe `admin-gitlab`.

## 2. Créer le fichier Docker Compose

Ouvrez le fichier Compose :

```bash
sudo nano /srv/gitlab/compose/docker-compose.yml
```

Ajoutez la configuration suivante :

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ce:19.2.0-ce.0
    container_name: gitlab
    hostname: git.example.com
    restart: unless-stopped
    stop_grace_period: 5m
    shm_size: "256m"

    volumes:
      - /srv/gitlab/config:/etc/gitlab
      - /srv/gitlab/data:/var/opt/gitlab
      - /srv/gitlab/logs:/var/log/gitlab

    ports:
      - "8080:80"
      - "2222:22"

    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url "http://git.example.com:8080"
        nginx['listen_port'] = 80
        nginx['listen_https'] = false
        letsencrypt['enable'] = false
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
```

Adaptez ces deux valeurs à votre nom d'hôte :

- `hostname` est le nom d'hôte interne du conteneur ;
- `external_url` est l'adresse publique utilisée par GitLab pour générer les liens et les URL de clonage.

Pour un homelab local, vous pouvez par exemple utiliser `git.homelab.local` à la place de `git.example.com` :

```yaml
hostname: git.homelab.local
external_url "http://git.homelab.local:8080"
```

Le nom indiqué dans `external_url` doit être résolu par le navigateur et le client Git. Il n'est pas nécessaire d'utiliser un DNS public pour un test : vous pouvez utiliser un DNS local ou ajouter une entrée dans le fichier `/etc/hosts` de chaque poste client.

Par exemple, sur un poste Linux, ajoutez une ligne contenant l'adresse IP de votre serveur :

```text
ADRESSE_IP_DU_SERVEUR git.example.com
```

Pour le choix homelab, utilisez à la place :

```text
ADRESSE_IP_DU_SERVEUR git.homelab.local
```

### Volumes

Les volumes conservent les données en dehors du conteneur :

- `/srv/gitlab/config` contient la configuration de GitLab ;
- `/srv/gitlab/data` contient les données de l'application et les dépôts Git ;
- `/srv/gitlab/logs` contient les journaux.

Ils permettent de conserver l'installation lors de la recréation ou de la mise à jour du conteneur.

### Ports

Le mappage `8080:80` rend l'interface web de GitLab disponible sur le port `8080` du serveur.

Le mappage `2222:22` rend le service SSH GitLab disponible sur le port `2222` du serveur. Le port `22` reste donc disponible pour l'accès SSH au serveur Linux.

La ligne `gitlab_rails['gitlab_shell_ssh_port'] = 2222` indique à GitLab d'afficher ce port dans ses URL de clonage SSH.

### Configuration Omnibus

`GITLAB_OMNIBUS_CONFIG` est une variable propre à l'image GitLab. Elle permet de transmettre des réglages qui seraient normalement écrits dans le fichier `gitlab.rb`.

Dans cette installation, GitLab utilise son NGINX intégré en HTTP sur son port interne `80`. HTTPS et Let's Encrypt sont désactivés, car la sécurisation avec un reverse proxy sera traitée ultérieurement.

## 3. Démarrer GitLab

Placez-vous dans le dossier contenant le fichier Compose, puis vérifiez sa syntaxe :

```bash
cd /srv/gitlab/compose
docker compose config
```

Démarrez GitLab en arrière-plan :

```bash
docker compose up -d
```

Le premier démarrage peut prendre plusieurs minutes. GitLab initialise plusieurs services avant de devenir disponible.

Vous pouvez visualiser l'utilisation du processeur et de la mémoire pendant cette étape :

```bash
htop
```

Lorsque la charge est stabilisée, vérifiez l'état des composants GitLab :

```bash
docker compose exec gitlab gitlab-ctl status
```

Chaque composant doit indiquer l'état `run`.

## 4. Se connecter avec le compte root

Récupérez le mot de passe initial généré par GitLab :

```bash
docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password
```

Ouvrez ensuite l'interface web :

```text
http://git.example.com:8080
```

Connectez-vous avec l'utilisateur `root` et le mot de passe affiché.

> Le fichier contenant le mot de passe initial est supprimé automatiquement après un délai défini par GitLab. Conservez ce mot de passe de façon sécurisée le temps de créer un administrateur nominatif.

## 5. Retirer les avertissements de sécurité

Depuis l'espace d'administration, ouvrez `Admin` → `Settings` → `General`.

1. Dépliez `New user account restrictions`.
2. Décochez `Allow new user accounts`, puis enregistrez les modifications.
3. Dépliez `Web IDE`.
4. Décochez `Enable single origin fallback`, puis enregistrez les modifications.

Le premier réglage empêche les inscriptions publiques. Le second désactive le mode de compatibilité du Web IDE qui génère un avertissement de sécurité sur une instance nouvellement installée.

## 6. Créer les utilisateurs

Depuis l'administration GitLab, créez un compte administrateur nominatif, par exemple `gandalf`.

Connectez-vous avec ce nouveau compte afin de vérifier qu'il possède bien les droits d'administration, puis désactivez le compte `root`. Le compte `root` ne doit pas être utilisé au quotidien.

Créez ensuite un compte utilisateur standard, par exemple `yeswecode`. Il servira à créer et manipuler un premier dépôt Git.

## 7. Créer et tester un dépôt en HTTP

Connectez-vous à GitLab avec le compte `yeswecode`.

Créez un projet nommé `test` et cochez l'option permettant de l'initialiser avec un fichier README.

Sur votre poste de travail ou sur le serveur utilisé pour les tests, configurez l'identité Git :

```bash
git config --global user.name "yeswecode"
git config --global user.email "yeswecode@example.com"
```

Clonez le dépôt en HTTP, modifiez le README, créez un commit et envoyez-le sur GitLab :

```bash
git clone http://git.example.com:8080/yeswecode/test.git
cd test
nano README.md
git add README.md
git commit -m "Met à jour le README"
git push
```

Git demande les identifiants du compte `yeswecode` lors du clonage et du `push`.

> HTTP n'est utilisé ici que pour la démonstration. Ne transmettez pas vos identifiants sur une connexion HTTP non chiffrée depuis Internet.

## 8. Créer une clé SSH et l'ajouter à GitLab

Créez une clé ED25519 dans un fichier nommé spécifiquement pour cette instance, puis affichez la clé publique :

```bash
cd ..
ssh-keygen -t ed25519 -C "yeswecode@gitlab" -f ~/.ssh/id_ed25519_yeswecode_gitlab
cat ~/.ssh/id_ed25519_yeswecode_gitlab.pub
```

Dans GitLab, ouvrez votre avatar puis `Edit profile` → `Access` → `SSH keys`.

Collez le contenu de la clé publique et ajoutez-la à votre compte. Ne copiez jamais la clé privée, qui correspond au fichier sans l'extension `.pub`.

## 9. Tester le dépôt en SSH

Chargez la clé personnalisée dans l'agent SSH et testez l'authentification avec GitLab :

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_yeswecode_gitlab
ssh -T -p 2222 git@git.example.com
```

Lors de la première connexion, SSH peut demander de confirmer l'empreinte du serveur. Vérifiez-la avant de répondre `yes`.

Clonez ensuite le même projet avec son URL SSH, modifiez le README et envoyez un nouveau commit :

```bash
git clone ssh://git@git.example.com:2222/yeswecode/test.git test-ssh
cd test-ssh
nano README.md
git add README.md
git commit -m "Met à jour le README"
git push
```

Le message de bienvenue de GitLab, le clonage et le `push` confirment que l'accès SSH fonctionne.

## Arborescence finale

```text
/srv/gitlab/
├── compose/
│   └── docker-compose.yml
├── config/
├── data/
└── logs/
```

Les dossiers `config`, `data` et `logs` sont complétés par GitLab au fil du démarrage et de l'utilisation.

## Récapitulatif rapide des commandes

```bash
# Arborescence et droits
sudo mkdir -p /srv/gitlab/{compose,config,data,logs}
sudo groupadd admin-gitlab
sudo usermod -aG admin-gitlab "$USER"
newgrp admin-gitlab
sudo chown -R root:admin-gitlab /srv/gitlab
sudo chmod -R g+rwX /srv/gitlab
sudo find /srv/gitlab -type d -exec chmod g+s {} +

# Fichier Compose
sudo nano /srv/gitlab/compose/docker-compose.yml

# Démarrage et contrôle
cd /srv/gitlab/compose
docker compose config
docker compose up -d
docker compose exec gitlab gitlab-ctl status
docker compose exec gitlab grep 'Password:' /etc/gitlab/initial_root_password

# Identité Git et test HTTP
git config --global user.name "yeswecode"
git config --global user.email "yeswecode@example.com"
git clone http://git.example.com:8080/yeswecode/test.git

# Clé et test SSH
ssh-keygen -t ed25519 -C "yeswecode@gitlab" -f ~/.ssh/id_ed25519_yeswecode_gitlab
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_yeswecode_gitlab
ssh -T -p 2222 git@git.example.com
git clone ssh://git@git.example.com:2222/yeswecode/test.git test-ssh
```

## Points d'attention

- Utilisez une version précise de l'image GitLab plutôt que le tag `latest`.
- Sauvegardez les dossiers `/srv/gitlab/config`, `/srv/gitlab/data` et `/srv/gitlab/logs` avant une mise à jour importante.
- Les ports `8080` et `2222` doivent être autorisés par le pare-feu du serveur si l'accès se fait depuis un autre poste.
- Le port `2222` concerne uniquement GitLab SSH ; l'administration SSH du serveur conserve le port `22`.
- La commande `ssh-add` est nécessaire ici parce que la clé porte un nom personnalisé. Sans agent SSH ou fichier `~/.ssh/config`, SSH ne la sélectionne pas automatiquement.
- Cette configuration est volontairement sans HTTPS. Pour une exposition sur Internet, placez GitLab derrière un reverse proxy HTTPS et mettez à jour `external_url`.

## Aller plus loin

Une fois GitLab accessible, vous pouvez :

- placer l'instance derrière NGINX avec HTTPS ;
- créer un runner GitLab pour exécuter les jobs CI/CD ;
- configurer des sauvegardes régulières ;
- créer des groupes et organiser les projets par équipe ou par application.
