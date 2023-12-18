# Gestion du server SiERA

## Création de l'instance sur Oracle Cloud

Ce rendre sur le site de [OCI](https://www.oracle.com/cloud/sign-in.html) et entrer dans le tenancy siera. Ensuite, utiliser l'adresse et le mot de passe stocké sur le drive. Ensuite, menu -> intances -> create instance (bien choisir le compartment siera à gauche de l'écran). La configuration plus tard sera faite sur Oracle Linux 9. Il faut ensuite choisir la "shape" en puce Ampère de 4 coeurs et 24Gb de RAM (ressource disponible gratuitement à vie). Ensuite, il faut laisser les paramètres pour la carte réseau virtuelle et copier la clé publique ssh qui sera utilisé pour se connecter la première fois au serveur. Enfin, choisir une taille custom de la partition de boot et entrer 200Gb ainsi qu'une performance VCI maximum.

## Mise en place lors du premier lancement

Pour commencer nous allons mettre à jour les packages avec ```sudo yum update -y```

### Installer git

Simplement faire la commande: ```sudo yum install git -y```

### Installer Neovim (non obligatoire)

Afin d'avoir un éditeur de texte accessible par ssh (comme vscode) nous allons installer NeoVim. D'abord il faut installer le repos EPEL 9 avoir d'avoir accès à plus de paquets. Puis nous allons simplement installer le paquet.

```
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo yum install -y neovim python3-neovim
```

C'est bon! NeoVim est accessible avec la commande nvim. Ensuite, soit vous êtes très motivié et voulez écrire le fichier de configuration soit vous pouvez cloner ma configuration sur ce [github]() avec:

```
mkdir -p ~/.config/nvim/
cd ~/.config/nvim/
git clone git@github.com:Captainfl4me/nvim-config.git .
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
nvim
```
Une fois sur nvim on peut installer les plugins avec :PlugInstall. Puis, fermer et relancer NeoVim et faire :CHADdeps afin d'installer les dernières dépendances. Normalement maitenant tout les paquets doivent être prêts.

### Installer Docker CE

On peut refaire une vérification que les packages nécessaires sont installés avec:

```
sudo yum update -y
sudo yum install -y yum-utils
```

On ajoute le repos Docker à notre manageur: ```sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo```. Le nouveau repos devrait apparaitre avec ```sudo dnf repolist```.

Enfin, on peut installer les paquets:

```
sudo yum install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker && sudo systemctl enable docker
sudo groupadd docker && sudo usermod -aG docker $USER
```

### Lancer le conteneur du site web

Nous allons d'abord ouvrir les ports 80, 443 et 5050 pour pouvoir accéder à ces ports depuis l'exterieur du réseau.

```
sudo firewall-cmd --zone=public --add-port=80/tcp --permanent
sudo firewall-cmd --zone=public --add-port=443/tcp --permanent
sudo firewall-cmd --zone=public --add-port=5050/tcp --permanent
sudo firewall-cmd --reload
```

Une fois les installations finalisé on peut copier ce repos dans le dossier de l'utilisateur et créer les networks necéssaires:

```
cd ~ && git clone ..
docker network create pterodactyl-network
```

Ensuite, nous allons charger la configuration pour récupérer les certificats SSL afin de lancer le site en HTTPS. Pour cela il faut ouvrir le fichier website/docker-compose.yml et vérifier que dans les volumes du services nginx les deux lignes sont bien présente comme ça: 

```yml
volumes:
    - ${PWD}/nginx/conf/only-certbot.conf:/etc/nginx/conf.d/default.conf
    #- ${PWD}/nginx/conf/https-full.conf:/etc/nginx/conf.d/default.conf
    - ./certbot/conf:/etc/nginx/ssl
    - ./certbot/data:/var/www/html
```

Si cela est bon alors on peut lancer le serveur une première fois avec ces lignes de commandes:

```
cd ~/siera-server/website && docker compose up
```

Il faut attendre que vous voyez ces lignes:

```
certbot    | Successfully received certificate.
certbot    | Certificate is saved at: /etc/letsencrypt/live/protoweb.siera-estaca.fr/fullchain.pem
certbot    | Key is saved at:         /etc/letsencrypt/live/protoweb.siera-estaca.fr/privkey.pem
certbot    | This certificate expires on 2024-03-15.
certbot    | These files will be updated when the certificate renews.
```

Cela signifie que les certificats ont bien été créés, vous pouvez terminer la commande en faisant un CTRL+C. Maintenant, on retourne dans le fichier docker-compose.yml ouvert précédemment et on change les lignes de volumes du service nginx ce cette façon afin de charger la configuration complète (il n'y aura plus besoin de modifier ça).


```yml
volumes:
    #- ${PWD}/nginx/conf/only-certbot.conf:/etc/nginx/conf.d/default.conf
    - ${PWD}/nginx/conf/https-full.conf:/etc/nginx/conf.d/default.conf
    - ./certbot/conf:/etc/nginx/ssl
    - ./certbot/data:/var/www/html
```

Enfin, faites les commandes suivantes:

```
cd ~/siera-server/website
docker compose down && docker compose up -d
```

Une fois cette commande effectué le serveur devrait être accessible sur le domaine de la siera en https. Il ne reste plus qu'à configurer le site wordpress.

### Installation de Pterodactyl

Pour cela il suffit de faire les deux commandes suivantes:

```
cd ~/siera-server/pterodactyl/panel
docker compose up -d
```

Nous allons maitenant créer un utilisateur. Il faut suivre les instructions dans le terminal. (Au moment de la mise en place du mot de passe il est normal que le terminal n'affiche rien pour le masquer).

```
cd ~/siera-server/pterodactyl/panel && docker-compose run --rm panel php artisan p:user:make
```

Le serveur Pterodactyl est maitenant disponible sur le nom de domaine au port 8080. 

## Gérer les sauvegardes du site

Pour gérer les sauvegardes on utilise l'extension Backup Migration.

