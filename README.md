## Gentoo Install
My setup for a clean Gentoo install on MY machine.


### Preparing the disks.
Formatting & mounting the partitons.
```
mkfs.vfat -F 32 /dev/nvme0n1p1 && mkswap /dev/nvme0n1p2 && mkfs.xfs /dev/nvme0n1p3
```
then
```
mkdir -p /mnt/gentoo/efi && mount /dev/nvme0n1p3 /mnt/gentoo && swapon /dev/nvme0n1p2
```
### Installing the tarball.
Getting and extracting the tarball.
```
cd /mnt/gentoo && wget https://distfiles.gentoo.org/releases/amd64/autobuilds/20250831T170358Z/stage3-amd64-openrc-20250831T170358Z.tar.xz && tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo
```
Cloning the repo.
Here I clone my make.conf and my kernel .config and another little confs.
```
git clone https://github.com/jesusferman/gentoo.git
```
### Installing the base system.
Mounting necessary filesystems.
```
mount --types proc /proc /mnt/gentoo/proc && mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys && mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev && mount --bind /run /mnt/gentoo/run && mount --make-slave /mnt/gentoo/run
```
```
test -L /dev/shm && rm /dev/shm && mkdir /dev/shm && mount --types tmpfs --options nosuid,nodev,noexec shm /dev/shm && chmod 1777 /dev/shm && chroot /mnt/gentoo /bin/bash
```
Preparing for the bootloader.
```
mount /dev/nvme0n1p1 /efi
```
Configuring portage.
```
emerge-webrsync
emerge --ask --verbose --oneshot app-portage/mirrorselect
mirrorselect -i -o >> /etc/portage/make.conf
emerge -q --sync

```
Selecting profiles.
```
eselect profile list
eselect profile set 1
```
Generating locales.
```
echo en_US ISO-8859-1
en_US.UTF-8 UTF-8 >> /etc/locale.gen
eselect locale list
eselect locale set
locale-gen
```
### Configuring the kernel
Installing and signing packages.
```
cp gentooinstall/usr/src/linux/.config /usr/src/linux/.config
emerge --ask sys-kernel/linux-firmware sys-kernel/installkernel
mkdir /pem && openssl req -new -noenc -utf8 -sha256 -x509 -outform PEM -out /pem/kernel_key.pem -keyout /pem/kernel_key.pem && chown root:root /pem/kernel_key.pem && chmod 400 /pem/kernel_key.pem
```

From here I could just follow along with https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/System
