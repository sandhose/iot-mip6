Setup du raspberry
==================

## Installation

Téléchargement de l'image 64 bits d'Ubuntu Server ici : https://ubuntu.com/download/raspberry-pi.

Installation de l'image sur la carte micro-SD :
- Depuis Windows : utilisation de Etcher (https://www.balena.io/etcher/)
- Depuis Linux : double clic sur l'image, et laisser l'utilitaire de GNOME restaurer l'image.


## Premier démarrage

Trouver l'adresse IP :
- lien local IPv6 en inspectant le traffic avec WireShark
- IPv4 si on se branche sur une box par exemple (avec DHCP)

Connexion en SSH (par défaut, l'image Ubuntu Server a un serveur SSH intégré, identifiants par défaut : `ubuntu` / `ubuntu`) sur l'IP trouvée.

Faire les mises à jour :
- mettre à jour le cache de `apt` : `sudo apt update`,
- mettre à jour tous les paquets : `sudo apt upgrade`
  Nécessite qu'Ubuntu finisse de faire toutes les mises à jour de sécurité, ce qui peut prendre un peu de temps.
  Pour suivre l'upgrade, on peut consulter les logs : `tail -f /var/log/unattended-upgrades/*`.

Lancer le playbook Ansible pour configurer le réseau et tous les paquets nécessaires.

## Récupérer UMIP

Récupérer en local : ftp://ftp.linux-ipv6.org/pub/usagi/patch/mipv6/umip-0.4/daemon/tarball/mipv6-daemon-umip-0.4.tar.gz (car directement depuis le raspberry on a eu quelques difficultés).
Le transférer sur les deux rasp avec `scp`.
L'extraire avec `tar xzvf mipv6-daemon-umip-0.4.tar.gz` et se rendre dans le dossier extrait : `cd mipv6-daemon-umip-0.4/`.

Lancer `sudo modprobe configs` pour générer la la configuration du kernel dans `/proc/config.gz`.

En lançant `bash ./chkconf_kernel.sh` (sans `bash`, on a plusieurs erreurs en plus), pour voir si on peut builder UMIP sans soucis.

On obtient la sortie suivante :

```
Using /proc/config.gz
 Warning: CONFIG_EXPERIMENTAL should be set to y (not supported)
 Warning: CONFIG_IPV6_MIP6 should be set to y (m)
 Warning: CONFIG_XFRM_USER should be set to y (m)
 Warning: CONFIG_XFRM_SUB_POLICY should be set to y (not supported)
 Warning: CONFIG_INET6_XFRM_MODE_ROUTEOPTIMIZATION should be set to y (not supported)
 Warning: CONFIG_IPV6_TUNNEL should be set to y (m)
 Warning: CONFIG_INET6_ESP should be set to y (m)
 Warning: CONFIG_NET_KEY should be set to y (m)
 Warning: CONFIG_NET_KEY_MIGRATE should be set to y (not supported)
```

Ceux marqués avec `(m)` sont disponibles sous forme de modules.

Il faut maintenant builder les paquets pour le nouveau kernel avec les options qu'il faut.


Noyau
=====

## Installation

Installer Ubuntu Server 19.10, puis certains paquets avec :

```sh
sudo apt install debootstrap qemu qemu-user-static binfmt-support
```

Vérifier les formats d'exécutables pris en charge avec : `sudo update-binfmts --display`.

Créer un système arm64 minimal dans lequel on va `chroot` et installer `vim` avec :

```sh
sudo debootstrap --variant=buildd --arch arm64 eoan /var/chroot/eoan http://ports.ubuntu.com
sudo chroot /var/chroot/eoan/
apt update && apt install vim
```

Éditer `/etc/apt/sources.list` pour qu'il contienne ceci :

```
deb http://ports.ubuntu.com eoan main universe restricted
deb-src http://ports.ubuntu.com eoan main universe restricted

deb http://ports.ubuntu.com eoan-updates main universe restricted
deb-src http://ports.ubuntu.com eoan-updates main universe restricted
```

Lancer `apt update`.

- cd /usr/local/src/
- mkdir builder
- chown _apt builder
- cd builder
- apt source linux-image-5.3.0-1012-raspi2
- cd linux-raspi2-5.3.0/
- apt build-dep linux-image-5.3.0-1012-raspi2
- apt install libncurses-dev flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf gcc-arm-linux-gnueabihf
- export LC_ALL=C
- chmod a+x debian/scripts/* debian/scripts/misc/*
- ./debian/rules editconfigs
Â» Do you want to edit config: armhf/config.flavour.raspi2? [Y/n] n
Â» Do you want to edit config: arm64/config.flavour.raspi2? [Y/n] y
Networking support ->
  Networking options ->
    PF_KEY MIGRATE
    Transformation sub policy support

- dpkg-buildpackage -a arm64



récupérer les fichiers .deb qui se trouvent désormais dans le dossier parent
(`scp "ludovic@metis:/var/chroot/eoan/usr/local/src/builder/*.deb" .)


les balancer sur le rasp

installer les deb: `dpkg -i *.deb`

fixer les versions avec :

```sh
sudo apt-mark hold \
  linux-buildinfo-5.3.0-1012-raspi2 \
  linux-modules-5.3.0-1012-raspi2 \
  linux-tools-5.3.0-1012-raspi2 \
  linux-headers-5.3.0-1012-raspi2 \
  linux-raspi2-headers-5.3.0-1012 \
  linux-image-5.3.0-1012-raspi2 \
  linux-raspi2-tools-5.3.0-1012
```

Et rebooter les raspberry.

Lancer à nouveau : `sudo modprobe configs` et `bash ./chkconf_kernel.sh`.

On obtient la sortie suivante :

```
Checking kernel configuration...
Using /proc/config.gz
 Warning: CONFIG_EXPERIMENTAL should be set to y (not supported)
 Warning: CONFIG_IPV6_MIP6 should be set to y (m)
 Warning: CONFIG_XFRM_USER should be set to y (m)
 Warning: CONFIG_INET6_XFRM_MODE_ROUTEOPTIMIZATION should be set to y (not supported)
 Warning: CONFIG_IPV6_TUNNEL should be set to y (m)
 Warning: CONFIG_INET6_ESP should be set to y (m)
 Warning: CONFIG_NET_KEY should be set to y (m)

Above 7 options may conflict with MIPL.
If you are not sure, use the recommended setting.
```

Le diff suivant https://github.com/torvalds/linux/commit/4c145dce26013763490df88f2473714f5bc7857d#diff-1108143834f27094c161049925222ab6 nous montre que `CONFIG_INET6_XFRM_MODE_ROUTEOPTIMIZATION` est désormais disponible sous le flag `CONFIG_IPV6`.

Le flag `CONFIG_EXPERIMENTAL` n'est pas nécessaire.

Il suffit de charger les modules restants avec : 

```sh
sudo modprobe mip6
sudo modprobe af_key
sudo modprobe esp6
sudo modprobe xfrm_user
sudo modprobe ip6_tunnel
sudo modprobe ip6_vti
```

Installer les paquets suivants, avec : `sudo apt install build-essential automake flex bison indent`

Une fois dans le dossier `mipv6-daemon-umip-0.4`, patcher UMIP avec notre patch `umip.patch` qui aura été déposé quelque part sur le raspberry qu'on applique avec `patch -p1 < CHEMIN_VERS/umip.patch`.

Lancer ensuite les commandes suivantes pour installer UMIP :

```sh
rm config.guess configure # pour forcer automake à les recréer
autoreconf
automake -a
./configure --with-builtin-crypto
make -j4 # pour utiliser les 4 coeurs CPU
sudo make install
```

Si tout est bon, on devrait pouvoir afficher l'aide d'UMIP et obtenir quelque chose de similaire :

```sh
> mip6d --help

Usage: mip6d [options]
Options:
  -V, --version            Display version information and copyright
  -?, -h, --help           Display this help text
  -c <file>                Read configuration from <file>

 These options override values read from config file:
  -d <number>              Set debug level (0-10)
  -l <file>                Write debug log to <file> instead of stderr
  -C, --correspondent-node Node is CN
  -H, --home-agent         Node is HA
  -M, --mobile-node        Node is MN

For bug reporting, see URL:http://www.mobile-ipv6.org/bugs/.
```
