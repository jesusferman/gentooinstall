# Gentoo Install.

## Preparing the disks.
Formating and mounting the /mnt.
```
mkfs.xfs -f /dev/nvme0n1p4
```
LUKS2 encryption setup.
```
cryptsetup luksFormat --type luks2 /dev/nvme0n1p4
```
```
cryptsetup luksOpen /dev/nvme0n1p4 root
```
```
mkfs.xfs -f /dev/mapper/gentoo && mount --mkdir /dev/mapper/gentoo /mnt/gentoo
```


## Stage 3 tarball.
Getting and extracting the tarball.
```
cd /mnt/gentoo && wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20250914T170345Z/stage3-amd64-openrc-20250914T170345Z.tar.xz && tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo && rm stage3-*
```

Cloning portage, kernel and DNS resolver.
```
git clone https://github.com/jesusferman/gentooinstall.git
```
```
cp -L /etc/resolv.conf /mnt/gentoo/etc/
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

Mounting /boot & /boot/efi.
```
mount /dev/nvme0n1p2 /boot && mount --mkdir /dev/nvme0n1p1 /boot/efi
```

Configuring Portage.
```
cp -rf gentooinstall/etc /mnt/gentoo/
```
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
mkdir -p /etc/keys && openssl req -new -noenc -utf8 -sha256 -x509 -outform PEM -out /etc/keys/kernel.pem -keyout /etc/keys/kernel.pem
```
```
openssl x509 -in /etc/keys/kernel.pem -inform PEM -out /etc/keys/kernel.der -outform DER
```
```
openssl req -new -noenc -utf8 -sha3-512 -x509 -outform PEM -out /etc/keys/modules.pem -keyout /etc/keys/modules.pem
```
```
chown root:root /etc/keys/* && chmod 400 /etc/keys/*
```


## Configuring the kernel
Installing firmware and dracut.
```
emerge -qa sys-kernel/linux-firmware sys-kernel/installkernel
```

Kernel configuration and compilation.
```
USE="experimental" emerge sys-kernel/gentoo-sources
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
sbsign /usr/src/linux-6.12.41-gentoo/arch/x86/boot/bzImage --cert /etc/keys/kernel.pem --key /etc/keys/kernel.pem --output /usr/src/linux-6.12.41-gentoo/arch/x86/boot/bzImage
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
emerge -q sys-fs/dpsfstools sys-fs/xfsprogs sys-block/io-scheduler-udev-rules
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
USE="persist" emerge -q app-admin/doas
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
