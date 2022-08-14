Gentoo
---

### mídia de boot
você pode usar o ssh (secure shell) na mídia de boot para mais facilidade de ~~copiar e colar~~
```
ifconfig
rc-service sshd start
```

### configuração dos discos
```
lsblk # mude /dev/sda para qualquer coisa que queira usar
parted -a optimal /dev/sda  # criando as partições
>----->
>mklabel gpt
>print  # rm 2
>unit mib

>mkpart primary 1 3
>name 1 grub
>set 1 bios_grub on

>mkpart primary 3 153
>name 2 boot
>set 2 boot on

>mkpart primary 153 653  # isso seta 500mb de swap
>name 3 swap

>mkpart primary 653 -1
>name 4 rootfs

>print
>quit
>-----<

# formatando as partições
mkfs.vfat /dev/sda1
mkfs.vfat /dev/sda2
mkswap /dev/sda3
swapon /dev/sda3
mkfs.ext4 /dev/sda4

mount /dev/sda4 /mnt/gentoo
cd /mnt/gentoo
```

### baixando o stage3
```
links https://gentoo.osuosl.org/releases/amd64/autobuilds/  # ponto significante de falha
>----->
>[most recent YYYYMMDD]
>stage3-*.tar.xz  # sem ser systemd ou nomultilib
>[save]
>[ok]
>q
>[Yes]
>-----<

tar xpvf stage3* --xattrs-include=`*.*` --numeric-owner  # extrair o tarball
```

### preparando o emerge
```
vi /mnt/gentoo/etc/portage/make.conf
>----->
CHOST="x86_64-pc-linux-gnu"
COMMON_FLAGS="-02 -pipe -march=native"

MAKEOPTS="-j12"  # 2gb de ram é necessário por thread. coloque o numero de threads que se encaixe nesse requerimento
#PORTAGE_NICENESS=1  # adicione isso quando terminar a instalaçao, isso prioritiza tarefas
ACCEPT_LICENSE="*"

USE="X nvidia"  # para usuários nvidia
GRUB_PLATFORMS="efi-64"
VIDEO_CARDS="nvidia"  # para usuários nvidia
>-----<
# aperte espaço para selecionar servidores da sua região. aperte enter para salvar e sair.
mirrorselect -io >> /mnt/gentoo/etc/portage/make.conf  # ponto significante de falha
mkdir /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf

cp -L /etc/resolv.conf /mnt/gentoo/etc/  # -L is --dereference
```

### montando o ambiente
```
# ponto significante de falha
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --rbind /run /mnt/gentoo/run
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/dev
mount --make-slave /mnt/gentoo/run

chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

mount /dev/sda2 /boot
```

### emergindo (dando emerge)
eu (wncry) listei os tempos de emerge do meu r5 2600 (ryzen 5 2600)
```
emerge-webrsync  # 30s
emerge --sync  # 1m

eselect news read  # man news.eselect (para ler o manual)

eselect profile list
eselect profile set 1  # se você planeja instalar o GNOME ou o KDE/Plasma, selecione seus respectivos perfis que incluem /desktop/ (não use systemd)

emerge -aqDNu @world  # 7m - 90m dependendo do perfil selecionado
```

### configurações adicionais
```
ls /usr/share/zoneinfo
echo "America/undisclosed_canadian_city_eh" > /etc/timezone
emerge --config timezone-data
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
locale-gen

eselect locale list
eselect locale set 4

env-update
source /etc/profile
export PS1="(chroot) ${PS1}"
```

### configurações gerais
```
emerge -aq gentoo-sources pciutils genkernel linux-firmware netifrc sysklogd vim  # instalar packages necessárias
ls -l /usr/src/linux*
mv /usr/src/linux* /usr/src/linux

vim /etc/fstab
>----->  # (wncry: por favor use tab pq fica muito bonito :3)
/dev/sda2	/boot	vfat	   defaults	   0 2
/dev/sda3	none	swap	   sw		      0 0
/dev/sda4	/	   ext4	   noatime 	   0 1
>-----<

genkernel all  # 30m (minutos)

ls /boot/vmlinu* /boot/initramfs*

# blkid
# vim /etc/fstab
# >----->
# # em caso de multiplos discos, coloque o UUID do sda2 anterior aqui
# >-----<

vim /etc/conf.d/hostname  # user@gentoo:$
>----->
hostname="gentoo"
>-----<

# o comando `ip a` nesse estágio não foi testado
ip a # enp4s0 é o nome do meu (wncry) dispositivo de internet (pode variar!)

vim /etc/conf.d/net
>----->
config_enp4s0="dhcp"  # coloque o nome do seu dispositivo de internet no lugar do enp4s0 (caso for diferente)
>-----<

cd /etc/init.d
ln -s net.lo net.enp4s0  # coloque o nome do seu dispositivo de internet aqui
rc-update add net.enp4s0 default  # coloque o nome do seu dispositivo de internet aqui

vim /etc/hosts
>----->  # (wncry: denovo, use tabs :3)
127.0.0.1	gentoo localhost
>-----<

vim /etc/security/passwdqc.conf
>----->
min=1,1,1,1,1  # isso é para você ter mais segurança :D
>-----<
passwd

date
vim /etc/conf.d/hwclock
>----->
EST  # seu fuso horário (brasil: gmt)
>-----<

rc-update add sysklogd default
```

### boot loader
```
emerge -aq e2fsprogs dosfstools dhcpcd grub:2
# exclua "--removable" CASO você NÃO esteja instalando em um dispositivo removível
grub-install --target=x86_64-efi --efi-directory=/boot  #--removable
grub-mkconfig -o /boot/grub/grub.cfg

exit
cd
umount -l /mnt/gentoo/dev/shm
umount -l /mnt/gentoo/dev/pts
umount -R /mnt/gentoo
reboot
```

### adicionando usuários e limpando
```
cd /
useradd -G users,wheel,video -m wncry
passwd wncry
visudo

rm /stage3-*.tar.xz
```

### ssh
```
emerge -aq openssh

rc-update add sshd default
rc-service sshd start

vim /etc/ssh/sshd_config
>----->
PermitRootLogin yes
>-----<
```

### nvidia drivers
onde a *diversão* começa...
```
emerge -aq nvidia-drivers xorg-server

env-update && source /etc/profile
gpasswd -a root video

cd /usr/src/linux && make menuconfig
>----->
>Device Drivers --->
>   Graphics support --->
>      < >  Nouveau (NVIDIA) cards
>-----<

mkdir /etc/X11/xorg.conf.d
vim /etc/X11/xorg.conf.d/nvidia.conf
>----->
Section "Device"
   Identifier  "nvidia"
   Driver      "nvidia"
EndSection
>-----<

emerge @module-rebuild
lsmod | grep nvidia  # talvez precise reiniciar
rmmod nvidia_drm
rmmod nvidia_modeset
rmmod nvidia
modprobe nvidia
modprobe nvidia_modeset
modprobe nvidia_drm
emerge -aqDNu @world  # reconstrói pacotes que se beneficiam das USE flags
etc-update  # atualiza pacotes

emerge -aq mesa-progs  # testando
glxinfo | grep direct
```

### display manager
#### ***!! esta sessão ainda está sobre refinamentos !!***
https://github.com/fairyglade/ly  # antigo
https://github.com/Cavernosa/ly  # fork do systemctl
```
git clone --recurse-submodules https://github.com/cavernosa/ly  # https://github.com/fairyglade/ly
cd ly
make && make run
make install installopenrc
rc-update add ly default
rc-service ly start
```

### desktop environment + windows manager (ambiente de trabalho + gerenciador de janelas)
#### gnome (não testado)
https://wiki.gentoo.org/wiki/GNOME/Guide
```
eselect profile set default/linux/amd64/17.1/desktop/gnome
emerge -aq gnome

env-update && source /etc/profile

rc-update add elogind boot
rc-service elogind start 

# para o display manager do gnome (gdm):
emerge -aqn gui-libs/display-manager-init
vim /etc/conf.d/display-manager
>----->
DISPLAYMANAGER="gdm"
>-----<

rc-update add display-manager default 
rc-service display-manager start 
systemctl enable gdm
systemctl start gdm
```

#### KDE/Plasma (não testado)
https://wiki.gentoo.org/wiki/KDE
https://wiki.gentoo.org/wiki/SDDM
```
eselect profile set default/linux/amd64/17.1/desktop/plasma

emerge -aq plasma-meta
vim ~/.xinitrc
>----->
#!/bin/sh
exec dbus-launch --exit-with-session startplasma-x11
>-----<

# para o "simple desktop display manager" (o plasma precisa dele)
emerge -aq sddm
usermod -a -G video sddm

vim /etc/sddm.conf
>----->
[X11]
DisplayCommand=/etc/sddm/scripts/Xsetup
>----->

mkdir -p /etc/sddm/scripts

vim /etc/sddm/scripts/Xsetup
>----->
setxkbmap us
>-----<

chmod a+x /etc/sddm/scripts/Xsetup

emerge -aq display-manager-init

vim /etc/conf.d/display-manager
>----->
CHECKVT=7
DISPLAYMANAGER="sddm"
>-----<

rc-update add display-manager default
rc-service display-manager start
```

#### para o ratpoison
https://wiki.gentoo.org/wiki/Ratpoison
```
emerge -aq ratpoison alacritty ranger htop dev-vcs/git feh display-manager-init

#cp /etc/X11/sessions/ratpoison.desktop /usr/share/xsessions/
vim .ratpoisonrc
>----->
exec feh --bg-scale ~/.wallpaper.png  # sets a wallpaper
exec /usr/bin/rpws init 4 -k  # opens four environments
exec ratpoison -c "banish"  # moves mouse to the bottom left corner

set border 3  # posicionamento de telas

# execução de aplicativos
bind a exec alacritty  # executa com 'ctrl+t a'
bind l exec librewolf

# gerenciamento de janelas
bind M-R remove
bind R resize  # executa com 'ctrl+t shift+r'
bind V vsplit
bind H hsplit

# ambientes
bind 1 exec rpws 1  # executed by 'ctrl+t 1'
bind 2 exec rpws 2
bind 3 exec rpws 3
bind 4 exec rpws 4
>-----<
```

## librewolf
```
vim /etc/portage/repos.conf/librewolf.conf
>----->
[librewolf]
priority = 50
location = /var/db/repos/librewolf
sync-type = git
sync-uri = https://gitlab.com/librewolf-community/browser/gentoo.git
auto-sync = Yes
>-----<

emaint -r librewolf sync
emerge -aq librewolf  # beard time - 65m
```

# VOCÊ ACABOU DE INSTALAR GENTOO :D
you can now tell others to 'install gentoo'
agora você pode dizer para outros 'instalarem gentoo'

traduzido por SynnK (aprendendo pt_BR)
