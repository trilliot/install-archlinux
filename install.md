Ce guide d'installation s'applique pour la configuration personnelle de mon ordinateur portable.

Inspiré de https://github.com/FredBezies/arch-tuto-installation/

# Sommaire

- Installation de ArchLinux

# Installation de ArchLinux

## Préambule

On part du principe que le support d'installation (clé USB ou DVD) est déjà créé.  
Le PC à installer doit être connecté à Internet.

## Système de base

### Avant d'aller plus loin

La première chose à faire est de configurer le clavier :

```
loadkeys fr-pc
```

### Partitionnement 

Déjà il faut savoir sur quel disque on va installer ArchLinux.  
Dans mon cas, il s'agit d'un disque NVMe. Il sera donc référencé sous `/dev/nvmeXnY`.

Lancer `cgdisk /dev/nvme0n1` (en remplaçant le chemin vers le disque au besoin) et partitionner comme suit :

| Partition | Size | Code |
| --------- | ---- | ---- |
| / | 50G | `8300` (Linux filesystem) |
| /boot | 1G | `8300` (Linux filesystem) |
| /boot/efi | 512M | `ef00` (EFI System) |
| /tmp | 2G | `8300` (Linux filesystem) |
| /var | 250G | `8300` (Linux filesystem) |
| /home | ∞ | `8300` (Linux filesystem) |

### Formattage

Il faut maintenant partionner nos partitions.

```
mkfs.ext4 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
mkfs.fat -F32 /dev/nvme0n1p3
mkfs.ext4 /dev/nvme0n1p4
mkfs.ext4 /dev/nvme0n1p5
mkfs.ext4 /dev/nvme0n1p6
```

Puis les monter :

```
mount /dev/nvme0n1p1 /mnt
mkdir /mnt/{boot,tmp,var,home}
mount /dev/nvme0n1p2 /mnt/boot
mount /dev/nvme0n1p4 /mnt/tmp
mount /dev/nvme0n1p5 /mnt/var
mount /dev/nvme0n1p6 /mnt/home
mkdir /mnt/boot/efi
mount /dev/nvme0n1p3 /mnt/boot/efi
```

### ArchLinux

#### OS de base

N'activer que les miroirs locaux dans `/etc/pacman.d/mirrorlist` pour accélérer l'installation.

Commencer par installer le système de base, quelques utilitaires (notamment pour la gestion de la batterie), et le bootloader UEFI :

```
pacstrap /mnt base base-devel
pacstrap /mnt zip unzip p7zip vim mc alsa-utils syslog-ng mtools dosfstools lsb-release exfat-utils bash-completion intel-ucode
pacstram /mnt grub efibootmgr
```

Générer le fichier /etc/fstab qui liste les partitions créées plus tôt :

```
genfstab -U -p /mnt >> /mnt/etc/fstab
```

On peut passer aux réglages dans l'OS, il faut d'abord rentrer dedans :

``` 
arch-chroot /mnt
```

#### Configuration

Première étape, changer le nom de la machine.

```
echo 'xps-de-teddy' > /etc/hostname
echo '127.0.1.1 xps-de-teddy.localdomain xps-de-teddy' >> /etc/hosts
```

Ensuite, on définit le fuseau horaire à utiliser et on synchronise l'horloge :

```
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc --utc
```

On va même installer *ntp* pour récupérer l'heure automatiquement :

```
pacman -S ntp
```

Puis cronie qui sert à exécuter des tâches récurrentes :

```
pacman -S cronie 
```

Vient ensuite la langue du système. Editer `/etc/locale.gen` et décommencer la ligne *fr_FR.UTF-8 UTF-8*.  
Puis générer les traductions et ajouter la nouvelle langue dans `/etc/locale.conf` et pour la session en cours :

```
locale-gen
echo 'LANG="fr_FR.UTF-8"' > /etc/locale.conf
export LANG=fr_FR.UTF-8
```

Enfin pour le clavier (oui, encore !), on va éditer le fichier `/etc/vconsole.conf` :

```
echo 'KEYMAP=fr' > /etc/vconsole.conf
```

Et voilà ! Nous sommes prêts à générer l'image du noyau :

```
mkinitcpio -p linux
```

Ne reste plus qu'à installer et configurer Grub dans l'UEFI :

```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

Pour paufiner et sécuriser, on va donner un mot de passe au compte super-utilisateur :

```
passwd root
```

Et également installer puis activer NetworkManager pour la gestion du réseau :

```
pacman -Syy networkmanager
systemctl enable NetworkManager.service
```

On active aussi quelques services utiles installés précédemment :

```
systemctl enable fstrim.timer
systemctl enable syslog-ng@default.service
systemctl enable cronie.service
systemctl enable ntpd.service
```

#### Finalisation

C'est prêt !  
Ne reste plus qu'à quitter le système, démonter nos volumes, et redémarrer la machine :

```
exit
umount -R /mnt
reboot
```

## Créer un compte utilisateur

On travaille depuis le début en tant que super-utilisateur. Il est préférable de se créer un compte utilisé personnel :

```
useradd -mG wheel -c 'Teddy Rilliot' -s /bin/bash trilliot
```

On lui donne ensuite un mot de passe :

```
passwd trilliot
```

Pour pouvoir utiliser les commandes super-utilisateur (_sudo_), il faut également éditer le fichier `/etc/sudoers`.  
Trouver la ligne _#Uncomment to allow members of group wheel to execute any command_ et décommentez la ligne en dessous.

A partir de maintenant, on peut arrêter d'utiliser le compte _root_ et passer sur son compte personnel.

## Installer yay

yay est un outil qui s'ajoute à pacman pour gérer les AUR.  
Il s'installe sous son compte personnel :

```
sudo pacman -S git
git clone https://aur.archlinux.org/yay
cd yay
makepkg -sri
```

Pour activer les couleurs, éditer le fichier `/etc/pacman.conf`.  
Chercher la ligne *#Color* et la décommenter.


## Installation de l'interface graphique

### Environnement de bureau XFCE

On commence par installer Xorg, et les paquets pour la gestion du clavier/souris/trackpad :

```
yay -S xorg-{server,xinit,apps} xf86-input-{mouse,keyboard,libinput} xdg-user-dirs
``` 

On ajoute le pilote pour la carte vidéo Intel (on s'occupera de la carte Nvidia plus tard).  
On installe également quelques polices :

```
yay -S xf86-video-intel ttf-{bitstream-vera,liberation,dejavu}
```

Vient ensuite la gestion des imprimantes et de leurs différents pilotes, que l'on active :

```
yay -S cups foomatic-{db,db-ppds,db-gutenprint-ppds,db-nonfree,db-nonfree-ppds} gutenprint nss-mdns
sudo systemctl enable avahi-daemon.service
sudo systemctl enable avahi-dnsconfd.service
sudo systemctl enable org.cups.cupsd.service
```

Et on active la résolution des noms MDNS dans le fichier `/etc/nsswitch.conf`, en ajoutant à la ligne, avant l'instruction *resolve* et *dns* :

```
hosts: ... mdns_minimal [NOTFOUND=return] ...
```

Après tous ces préparatifs, il est temps de passer à l'installation de XFCE :

```
yay -S xfce4 
```

On va aussi définir tout de suite l'agencement du clavier :

```
sudo localectl set-x11-keymap fr
```

Puis on ajoute des utilitaires de XFCE, et les modules pour la barre des tâches et l'explorateur de fichiers :

```
yay -S xfce4-{artwork,notifyd,screensaver,screenshooter,taskmanager}
yay -S xfce4-{battery,clipman,cpufreq,cpugraph,datetime,diskperf,fsguard,genmon,mailwatch,mount,mpc,netload,notes,pulseaudio,sensors,smartbookmark,systemload,timer,verve,wavelan,weather,whiskermenu,xkb}-plugin
yay -S thunar-{archive,media-tags}-plugin 
```

Le système de base est prêt, on peut maintenant ajouter des logiciels plus spécifiques :

```
yay -S firefox-i18n-fr firefox-ublock-origin vlc ffmpegthumbnailer evince xarchiver galculator pavucontrol pulseaudio-{alsa,bluetooth} blueman libcanberra-{pulse,gstreamer} system-config-printer network-manager-applet mugshot
```

On y ajoute différents pilotes systèmes de fichiers montables :

```
yay -S gvfs-{afc,google,gphoto2,mtp,nfs,smb}
```

Ainsi que des codecs audio et vidéo :

```
yay -S gst-plugins-{base,good,bad,ugly} gst-libav
```

### Gestionnaire d'affichage lightdm

Ne reste plus qu'à ajouter un gestionnaire d'affichage et un écran de connexion :

```
yay -S lightdm-slick-greeter
```

Il faut dire à *lightdm* quel écran de connexion utiliser en éditant le fichier `/etc/lightdm/lightdm.conf`.  
Trouver la ligne *#greeter-session=example-gtk-gnome* dans la section *[Seat:*]* et la modifier pour :

```
greeter-session=lightdm-slick-greeter
```

Et à démarrer l'interface graphique :

```
sudo systemctl start lightdm.service
```

Si tout se passe bien, on n'oublie pas de l'activer :

```
sudo systemctl enable lightdm.service
```

### Thème XFCE

Pour embellir l'interfae de bureau, on va installer un thème graphique et de nouvelles icônes :

```
yay -S qogir-gtk-theme-git qogir-icon-theme-git
```

Il faut ensuite activer le thème et les icônes dans *Apparence*.

### Gestion du touchpad

On configure le touchpad pour qu'il ait le même comportement que celui d'un MacBook (question d'habitude) :

```
sudo tee /etc/X11/xorg.conf.d/30-touchpad.conf <<EOF
Section "InputClass"
    Identifier "touchpad"
    Driver "libinput"
    MatchIsTouchpad "on"
    Option "ClickMethod" "clickfinger"
    Option "NaturalScrolling" "on"
    Option "ScrollMethod" "twofinger"
EndSection
EOF
```

On va également gérer les mouvements multi-touch. On commence par s'ajouter au groupe *input* :

```
sudo gpasswd -a $USER input
```

Il faut ensuite se déconnecter puis se reconnecter de sa session.  
On installe ensuite le paquet :

```
yay -S libinput-gestures
```

On peut si besoin personnaliser la configuration pour ajouter ou modifier des mouvements.  
Pour cela on copie le fichier de configuration dans son répertoire personnel :

```
cp /etc/libinput-gestures.conf ~/.config/
vim ~/.config/libinput-gestures.conf
```

Enfin on active le service :

```
libinput-gestures-setup autostart
```

### Ouverture du menu au clavier

Il n'y a pas de raccourci par défaut pour ouvrir le menu d'applications au clavier. On va en ajouter un.

Ouvrir l'application *Clavier* et dans l'onglet *Raccourcis d'applications*, ajouter sur le bouton *+ Ajouter*.  
La commande à exécuter est `xfce4-popup-whiskermenu`, il ne reste plus qu'à valider et appuyer sur la touche qui va déclencher l'action (dans mon cas *Super*).

## Optimisation

### Gestion de la batterie

*TODO: bbswitch bumblebee prime*

#### Mise en veille

Avant de commencer, il faut changer le mode de mise en veille (celui par défaut n'est pas efficient).  
On édite les paramètres du kernel dans `/etc/default/grub/` pour modifier la ligne :

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet mem_sleep_default=deep"
```

Ensuite, on redéploie le booloader Grub, puis on redémarre :

```
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --recheck
sudo grub-mkconfig -o /boot/grub/grub.cfg
reboot
```

La commande suivante devrait afficher le mode *deep* actif :

```
cat /sys/power/mem_sleep
```

#### Optimisation : TLP

Pour optimiser la consommation de l'ordinateur sur batterie, on installe *tlp* et son extension *tlp-rdw* (pour les cartes sans-fil) :

```
yay -S tlp tlp-rdw
```

On masque ensuite les services requis par *tlp-rdw*, et on active *NetworkManager-dispatcher* :

```
sudo systemctl mask systemd-rfkill.service systemd-rfkill.socket
sudo systemctl enable NetworkManager-dispatcher.service
```

Enfin on peut lancer *tlp* et *tlp-rdw* :

```
sudo systemctl enable tlp.service tlp-sleep.service
sudo systemctl start tlp.service tlp-sleep.service
```

Pour suivre la consommation de la batterie, on installe *powertop* :

```
yay -S powertop
```

### Carte graphique dédiée

*TODO*
