# Gestion du server SiERA

## Mise en place sur Oracle Cloud Infrastructure

### Création de l'instance

Ce rendre sur le site de [OCI](https://www.oracle.com/cloud/sign-in.html) et entrer dans le tenancy siera. Ensuite, utiliser l'adresse et le mot de passe stocké sur le drive. Ensuite, menu -> intances -> create instance (bien choisir le compartment siera à gauche de l'écran). La configuration plus tard sera faite sur Ubuntu 22.04. Il faut ensuite choisir la "shape" en puce Ampère de 4 coeurs et 24Gb de RAM (ressource disponible gratuitement à vie). Ensuite, il faut laisser les paramètres pour la carte réseau virtuelle et copier la clé publique ssh qui sera utilisé pour se connecter la première fois au serveur. Enfin, choisir une taille custom de la partition de boot et entrer 200Gb ainsi qu'une performance VCI maximum.

### Création des règles de réseaux

Pour modifier l'adresse IP et la rendre réservé il faut créer une adresse réservé avec: Networking->IP management-> Reserved public IPs et créer une adresse. Puis, il faut aller dans instances->NOM_DE_LINSTANCE->Attached VNICs->Primary VNIC->IPv4 Addresses->Primary IP->Edit. Puis choisir "no public IP" et mettre à jour et retourner dans "edit" et choisir "reserved public IP".
Pour modifier les ports accessible depuis l'exterieur du réseau il faut aller dans instances->NOM_DE_LINSTANCE->Attached VNICs->Primary VNIC->Subnet->Default Security List. Ensuite, "Add Ingress Rule" et remplir en source 0.0.0.0/0 choisir le protocol et le port de destination.
Ici il faut ouvrir les ports 80, 443, 8080 et 8443 en TCP.

## Mise en place lors du premier lancement

Pour commencer nous allons mettre à jour les packages avec ```sudo apt update && sudo apt ugrade -y```

### Installer git

Normalement git sera déjà installé. On peut le vérifier en faisant un ```git --version```.

Si jamais git n'est pas installé il faut simplement installer le paquet: ```sudo apt install -y git```

### Installer Neovim (non obligatoire)

Afin d'avoir un éditeur de texte accessible par ssh (comme vscode) nous allons installer NeoVim. D'abord il faut installer le repos EPEL 9 avoir d'avoir accès à plus de paquets. Puis nous allons simplement installer le paquet.

```
sudo apt install -y neovim python3-neovim python3-venv python3-pip
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

### Installer [Docker Engine](https://docs.docker.com/engine/install/ubuntu/)

On va ajouter le dépos docker à notre gestionnaire de paquet avec:

```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Enfin, on peut installer les paquets:

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker && sudo systemctl enable docker
sudo groupadd docker && sudo usermod -aG docker $USER
```

On pourra vérifier que docker est bien installé avec: ```docker run hello-world```

### Lancer le conteneur du site web

Nous allons copier ce repos dans le dossier de l'utilisateur et créer les networks necéssaires:

```
cd ~ && git clone git@github.com:Captainfl4me/siera-server.git
docker network create pterodactyl-panel
docker network create pterodactyl-wings
```

Ensuite, nous allons charger la configuration pour récupérer les certificats SSL afin de lancer le site en HTTPS. Pour cela il faut lancer le serveur avec uniquement le gestionnaire de certificats: 

```
cd ~/siera-server/website/ && docker compose -f docker-compose-only-cert.yml up
```

Il faut attendre que vous voyez ces lignes:

```
certbot    | Successfully received certificate.
certbot    | Certificate is saved at: /etc/letsencrypt/live/protoweb.siera-estaca.fr/fullchain.pem
certbot    | Key is saved at:         /etc/letsencrypt/live/protoweb.siera-estaca.fr/privkey.pem
certbot    | This certificate expires on 2024-03-15.
certbot    | These files will be updated when the certificate renews.
```

Cela signifie que les certificats ont bien été créés, vous pouvez terminer la commande en faisant un CTRL+C. Vous pouvez maintenant lancer le serveur avec les commandes suivantes: 

```
cd ~/siera-server/website
docker compose down && docker compose up -d
```

Une fois cette commande effectué le serveur devrait être accessible sur le domaine de la siera en https. Il ne reste plus qu'à configurer le site wordpress.

### Installation de Pterodactyl

> Pour la configuration et la création du tutoriel [lien utile](https://www.youtube.com/watch?v=_ypAmCcIlBE).

Pour cela il suffit de faire les deux commandes suivantes:

```
cd ~/siera-server/pterodactyl/panel
docker compose up -d
```

Nous allons maitenant créer un utilisateur. Il faut suivre les instructions dans le terminal. (Au moment de la mise en place du mot de passe il est normal que le terminal n'affiche rien pour le masquer).

```
cd ~/siera-server/pterodactyl/panel && docker compose run --rm panel php artisan p:user:make
```

Le serveur Pterodactyl est maitenant disponible sur le nom de domaine au port 8080. 

Ensuite, il faut se connecter et créer une "Locations". Puis, créer un "node" avec une connection SSL le paramètre "Behind Proxy" et le port de "Daemon Port" à 8443 et le "Deamon SFTP Port" à 2022. Remplacer le fichier de configuration (pterodactyl/wings/etc/config.yml) avec le texte généré dans l'onglet "Configuration".

Ensuite lancer le noeud wings avec la commande:

```
cd ~/siera-server/pterodactyl/wings && docker compose up -d
```

Le node devrait maintenant apparaitre avec un coeur vert devant.

Ensuite, il faut définir l'allocation des ports que va utiliser Wings pour créer les serveurs de jeu. Pour cela il faut rentrer l'IP du serveur et la fourchette de ports.
> Par exemple: IP adress = 141.145.199.141 et Ports=27000-27099

## Gérer les sauvegardes du site

Pour gérer les sauvegardes on utilise l'extension Backup Migration.

