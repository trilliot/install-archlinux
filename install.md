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

| Parititon | Size | Code |
| --------- | ---- | ---- |
| / | 20G | `8300` (Linux filesystem) |
| /boot/efi | 512M | `ef00` (EFI System) |
| /home | ∞ | `8300` (Linux filesystem) |

### Formattage

Il faut maintenant partionner nos partitions.

```
mkfs.ext4 /dev/nvme0n1p1
mkfs.fat -F32 /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
```

Puis les monter :

```
mount /dev/nvme0n1p1 /mnt
mkdir /mnt/{boot,boot/efi,home}
mount /dev/nvme0n1p2 /mnt/boot/efi
mount /dev/nvme0n1p3 /mnt/home
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

Vient après la langue du système. Editer `/etc/locale.gen` et décommencer la ligne *fr_FR.UTF-8 UTF-8*.  
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
systemctl enable NetworkManager
```

Le disque utilisé étant un NVMe, il faut aussi lancer le timer fstrim :

```
systemctl enable fstrim.timer
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

*TODO*
*XFCE ?*
*i3wm(-gaps) + polybar ?*

## Optimisation

### Gestion de la batterie

*TODO: tlp powertop bbswitch bumblebee prime*

#### Mise en veille

Avant de commencer, il faut changer le mode de mise en veille (celui par défaut n'est pas efficient).  
On édite les paramètres du kernel dans `/etc/default/grub/` pour modifier la ligne :

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet mem_sleep_default=deep"
```

Ensuite, on redéploie le booloader Grub, puis on redémarre :

```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --recheck
grub-mkconfig -o /boot/grub/grub.cfg
reboot
```

La commande suivante devrait afficher le mode *deep* actif :

```
cat /sys/power/mem_sleep
```

*TODO: tlp powertop bbswitch bumblebee prime*

### Carte graphique dédiée

*TODO*
