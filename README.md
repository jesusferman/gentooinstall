# Gentoo Install.

## Preparing the livemedia.
Cloning dnf, etc and kernel configs.
```
git clone https://github.com/jesusferman/gentooinstall.git
```
Installing codecs.
```
sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm && sudo dnf config-manager setopt fedora-cisco-openh264.enabled=1 && sudo dnf install libavcodec-freeworld
```
Installing a browser.
```
sudo dnf config-manager addrepo --from-repofile=https://repository.mullvad.net/rpm/stable/mullvad.repo && sudo dnf install mullvad-browser && sudo dnf remove firefox
```


## Preparing the disks.
Formating and mounting the /mnt.
```
mkfs.vfat -F32 /dev/nvme0n1p1 && mkswap /dev/nvme0n1p2 && mkfs.ext4 /dev/nvme0n1p3
```
LUKS2 encryption setup.
```
cryptsetup luksFormat --type luks2 /dev/nvme0n1p3
```
```
cryptsetup luksOpen /dev/nvme0n1p3 root
```
```
mkfs.ext4 /dev/mapper/gentoo && mount --mkdir /dev/mapper/gentoo /mnt/gentoo
```


## Stage 3 tarball.
Getting and extracting the tarball.
```
cd /mnt/gentoo && wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20250928T160345Z/stage3-amd64-openrc-20250928T160345Z.tar.xz && tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo && rm stage3-*
```
Setting portage and etc confs.
```
cp -L /gentooinstall/etc/ /mnt/gentoo/etc/
```


## Installing the base system.
Mounting and fixing filesystems.
```
mount --types proc /proc /mnt/gentoo/proc && mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys && mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev && mount --bind /run /mnt/gentoo/run && mount --make-slave /mnt/gentoo/run
```
```
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm && mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm && chmod 1777 /dev/shm
```
Change root into /mnt/gentoo.
```
chroot /mnt/gentoo /bin/bash
```
Mounting /boot/efi.
```
mount --mkdir /dev/nvme0n1p1 /boot/efi
```
Configuring Portage.
```
emerge-webrsync
```
```
getuto
```
```
emerge -q --sync
```
Configuring locales.
```
nano /etc/locale.gen
```
```
locale-gen
```
```
eselect locale set 3
```
```
nano /etc/env.d/02locale
```
```
ln -sf /usr/share/zoneinfo/ /etc/localtime
```
```
env-update && source /etc/profile
```
Generating signing keys.
```
mkdir -p /etc/keys && openssl req -new -noenc -utf8 -sha256 -x509 -outform PEM -out /etc/keys/kernel.pem -keyout /etc/keys/kernel.key
```
```
openssl x509 -in /etc/keys/kernel.pem -inform PEM -out /etc/keys/kernel.der -outform DER
```
```
openssl req -new -noenc -utf8 -sha3-512 -x509 -outform PEM -out /etc/keys/modules.pem -keyout /etc/keys/modules.key
```
```
chown root:root /etc/keys/* && chmod 700 /etc/keys && chmod 400 /etc/keys/*
```


## Configuring the kernel
Installing firmware and dracut.
```
emerge -qa sys-kernel/linux-firmware sys-kernel/installkernel
```
Kernel configuration and compilation.
```
emerge sys-kernel/gentoo-sources
```
```
cp gentooinstall/.config /usr/src/linux-6.12.41-gentoo/
```
```
eselect kernel set 1 && env-update && source /etc/profile
```
Set BT, ALSA, Virtio as modules.
```
make menuconfig
```
```
make -j12 && make -j12 modules_install
```
```
sbsign /usr/src/linux-6.12.41-gentoo/arch/x86/boot/bzImage --cert /etc/keys/kernel.pem --key /etc/keys/kernel.key --output /usr/src/linux-6.12.41-gentoo/arch/x86/boot/bzImage
```
```
make install
```
Any doubts here can, and should, be answered by the handbook.
https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Kernel#Manual_process


## Configuring the system
Creating the fstab.
```
blkid
```
```
nano /etc/fstab
```

Enabling udev and services.
```
emerge -q sys-fs/dosfstools sys-fs/e2fsprogs sys-block/io-scheduler-udev-rules
```
```
emerge -q net-misc/chrony app-admin/sysklogd net-wireless/iwd
```
```
rc-update add chronyd default
```
```
rc-update add sysklogd default
```
```
rc-update add iwd default
```
```
rc-update add sshd default
```

Configuring GRUB.
```
emerge -qa sys-boot/grub sys-boot/shim sys-boot/mokutil sys-boot/efibootmgr
```
```
cp /usr/share/shim/BOOTX64.EFI /boot/efi/EFI/Gentoo/shimx64.efi && cp /usr/share/shim/mmx64.efi /boot/efi/EFI/Gentoo/mmx64.efi && cp /usr/lib/grub/grub-x86_64.efi.signed /boot/efi/EFI/Gentoo/grubx64.efi
```
```
mokutil --import /etc/keys/kernel.der
```
```
efibootmgr --create --disk /dev/mapper/gentoo --part PARTUUID --loader '\EFI\Gentoo\shimx64.efi' --label 'GRUB via Shim' --unicode
```
```
grub-mkconfig -o /boot/efi/EFI/Gentoo/grub.cfg
```

Creating an user.
```
emerge -q app-shells/zsh app-shells/gentoo-zsh-completions
```
```
emerge -q app-admin/doas
```
```
groupadd docker && useradd -mG users,wheel,docker,video,audio -s /bin/zsh jesus
```
```
passwd jesus
```
```
su jesus
```
```
doas emerge app-misc/fastfetch
```

Exiting and rebooting.
```
exit
```
```
cd && umount -l /mnt/gentoo/dev{/shm,/pts,} && umount -R /mnt/gentoo
```
```
reboot
```
