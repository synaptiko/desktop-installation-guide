# Arch Linux Desktop installation guide
## Goals

- minimal system based on [i3wm](http://i3wm.org/) and few utilities
- [encrypted root partition](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system)
- power management fine-tuning for a laptop

## Decisions

- use systemd-boot instead of Grub
- use dm-crypt with LUKS and Ext4
- use autologin & avoid display manager (root partition is encrypted, one password is enough)
  - but use i3lock (or similar) for manual locking

## Preparation

- download the latest Arch Linux iso & sig file from https://www.archlinux.org/download/
- run `gpg2 --verify archlinux-YYYY-MM-01-dual.iso.sig archlinux-YYYY-MM-01-dual.iso`
- run `sudo imagewriter` (https://aur.archlinux.org/packages/imagewriter) or `Etcher` (https://etcher.io) and write the image to a USB flash disk

## Variants

- laptop (L) or desktop (D)
	- *laptop includes proper wifi setup and power management*
- encrypted root (R) or encrypted root+home (RH)

## Steps

- boot from USB flash drive to Arch Linux

### Wipe the disk(s)

- wipe /dev/sda (**R** & **RH** variant)

	cryptsetup open --type plain /dev/sda container --key-file /dev/random
	dd if=/dev/zero of=/dev/mapper/container bs=1M status=progress

- wipe /dev/sdb (**RH** variant) *can be done in parallel in the other VT*

	cryptsetup open --type plain /dev/sdb container --key-file /dev/random
	dd if=/dev/zero of=/dev/mapper/container bs=1M status=progress

### Prepare partitions

- prepare EFI boot & root partitions (**R** & **RH** variants)

	gdisk /dev/sda
		Command (? for help): o
		Command (? for help): n
		Partition number (1-128, default 1): 1
		First sector (34-1000215182, default = 2048) or {+-}size{KMGTP}: 2048
		Last sector (2048-1000215182, default = 1000215182) or {+-}size{KMGTP}: 512M
		Hex code or GUID (L to show codes, Enter = 8300): ef00
		Command (? for help): n
		Partition number (2-128, default 2): 2
		First sector (34-1000215182, default = 1050624) or {+-}size{KMGTP}: 1050624
		Last sector (1050624-1000215182, default = 1000215182) or {+-}size{KMGTP}: 1000215182
		Hex code or GUID (L to show codes, Enter = 8300): 8304
		Command (? for help): w

	# boot partition
	mkfs.fat -F32 /dev/sda1

	# root partition
	cryptsetup -y -v luksFormat /dev/sda2
	cryptsetup open /dev/sda2 cryptroot
	mkfs.ext4 /dev/mapper/cryptroot

- prepare home partition on separate disk (**RH** variant)

	gdisk /dev/sdb
		Command (? for help): o
		Command (? for help): n
		Partition number (1-128, default 1): 1
		First sector (34-1000215182, default = 2048) or {+-}size{KMGTP}: 2048
		Last sector (2048-1000215182, default = 1000215182) or {+-}size{KMGTP}: 1000215182
		Hex code or GUID (L to show codes, Enter = 8300): 8302
		Command (? for help): w

	# home partition
	cryptsetup -y -v luksFormat /dev/sdb1
	cryptsetup open /dev/sdb1 crypthome
	mkfs.ext4 /dev/mapper/crypthome

### Install basic system

- installing a laptop only with wifi connectivity? (**L** variant)

	wifi-menu

- setup NTP to have synchronized time

	timedatectl set-ntp true

- mount partitions

	mount /dev/mapper/cryptroot /mnt
	mkdir /mnt/boot
	mount /dev/sda1 /mnt/boot

- mount home (**RH** variant)

	mkdir /mnt/home
	mount /dev/mapper/crypthome /mnt/home

- install base system packages

	pacstrap -i /mnt base base-devel

- enter following choices (see [https://github.com/synaptiko/desktop-installation-guide/blob/master/README.md#includedexcluded-packages-explanation](Included/excluded packages explanation))
	- **base:** 1-5,7-20,22-25,27-28,30,32-37,40-47,49
	- **base-devel:** all

- generate fstab for /dev/sda (**R** & **RH** variant)
	
	echo -n > /mnt/etc/fstab
	echo -e "UUID="`blkid -o value -s UUID /dev/sda1`"\t/boot\tvfat\tdefaults\t0 1" >> /mnt/etc/fstab
	echo -e "/dev/mapper/cryptroot\t/\text4\trw,relatime,data=ordered,errors=remount-ro\t0 1" >> /mnt/etc/fstab

- generate fstab for /dev/sdb (**RH** variant)

	echo -e "/dev/mapper/crypthome\t/\text4\trw,relatime,data=ordered,errors=remount-ro\t0 2" >> /mnt/etc/fstab

- setup the system in the chrooted env (see [https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio](Encrypting an entire system/Configuring mkinitcpio))

	arch-chroot /mnt /bin/bash
	sed -i '/^#\(cs_CZ\|en_US\)\.UTF-8/s/^#//' /etc/locale.gen
	locale-gen
	echo LANG=en_US.UTF-8 > /etc/locale.conf
	ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime
	hwclock --systohc --utc
	sed -i '/^HOOKS=/s/".\+"/"systemd autodetect modconf keyboard block sd-encrypt filesystems fsck"/' /etc/mkinitcpio.conf
	mkinitcpio -p linux
	mkdir /boot/EFI
	bootctl --path=/boot install
	echo -n > /boot/loader/loader.conf
	echo -e "default\tarch" >> /boot/loader/loader.conf
	echo -e "timeout\t0" >> /boot/loader/loader.conf
	echo -e "editor\t0" >> /boot/loader/loader.conf
	pacman -S intel-ucode
	echo -n > /boot/loader/entries/arch.conf
	echo -e "title\tArch Linux" >> /boot/loader/entries/arch.conf
	echo -e "linux\t/vmlinuz-linux" >> /boot/loader/entries/arch.conf
	echo -e "initrd\t/intel-ucode.img" >> /boot/loader/entries/arch.conf
	echo -e "initrd\t/initramfs-linux.img" >> /boot/loader/entries/arch.conf
	echo -e "options\trd.luks.name="`cryptsetup luksUUID /dev/sda2`"=cryptroot root=/dev/mapper/cryptroot" >> /boot/loader/entries/arch.conf

- create crypttab (**RH** variant)

	echo -e "crypthome\tUUID="`cryptsetup luksUUID /dev/sdb1` > /etc/crypttab

- create the hostname file

	echo jprokop-<some-unique-short-machine-nickname> > /etc/hostname

- create new user

	pacman -S zsh
	useradd -m -g users -s /usr/bin/zsh jprokop
	gpasswd -a jprokop lp
	gpasswd -a jprokop wheel
	gpasswd -a jprokop network
	gpasswd -a jprokop video
	gpasswd -a jprokop audio
	gpasswd -a jprokop storage
	gpasswd -a jprokop uucp
	passwd jprokop

- configure sudo

	echo -n > /etc/sudoers.d/10-jprokop
	echo "jprokop ALL=(ALL) ALL" >> /etc/sudoers.d/10-jprokop
	echo "jprokop "`cat /etc/hostname`"= NOPASSWD: /usr/bin/poweroff, /usr/bin/reboot" >> /etc/sudoers.d/10-jprokop

- test sudo access

	sudo -s -u jprokop
	sudo pacman -Sy
	exit

- install network related packages (**L** & **D** variants)

	pacman -S networkmanager bind-tools openresolv openssh
	systemctl enable NetworkManager.service

- install network related packages (**L** variant)
  - *use wifi-menu/nmtui/nm-applet for wifi configuration after reboot*

	pacman -S crda wireless-regdb iw wpa_supplicant dialog network-manager-applet
	sed -i '/^#WIRELESS_REGDOM="CZ"/s/^#//' /etc/conf.d/wireless-regdom

- lock root account & reboot

	passwd -l root
	exit
	umount -R /mnt
	reboot

- login as the new user
- setup NTP to have synchronized time

	sudo timedatectl set-ntp true

- connect to wifi (**L** variant)

	sudo wifi-menu

### Install pacaur (prerequisite for .files deps installation)

	sudo pacman -S git
	cd
	mkdir Packages && cd Packages
	git clone https://aur.archlinux.org/cower.git
	cd cower
	gpg --recv-keys --keyserver hkp://pgp.mit.edu `grep validpgpkeys PKGBUILD | cut -d"'" -f2`
	makepkg -sirc
	cd ..
	git clone https://aur.archlinux.org/pacaur.git
	cd pacaur
	makepkg -sirc

### Configure system with .files

	cd
	git clone https://github.com/synaptiko/.files
	cd .files
	git remote remove origin
	git remote add origin git@github.com:synaptiko/.files.git
	./install-dependencies.sh
	./init.sh

### Setup basic firewall rules in GUFW

	sudo systemctl enable ufw
	sudo systemctl start ufw
	sudo ufw allow in ssh comment ssh
	sudo ufw reload

### Install TLP (**L** variant)

- see details [TLP FAQ - Battery](http://linrunner.de/en/tlp/docs/tlp-faq.html#battery)

	sudo pacman -S tlp acpi_call lsb-release smartmontools x86_energy_perf_policy ethtool
	sudo systemctl enable tlp.service
	sudo systemctl enable tlp-sleep.service
	sudo systemctl mask systemd-rfkill.service
	sudo systemctl mask systemd-rfkill.socket

### Reboot to the i3

	reboot

### Generate and use SSH keys

- generate ssh keys

	ssh-keygen -t rsa -b 4096 -C "jprokop@synaptiko.cz ("`cat /etc/hostname`")"
	cat ~/.ssh/id_rsa.pub | xsel -i -b

- add public key to your github & bitbucket account

## Included/excluded packages explanation

**base group:**

	1) bash  2) bzip2  3) coreutils  4) cryptsetup  5) device-mapper  6) dhcpcd  7) diffutils
	8) e2fsprogs  9) file  10) filesystem  11) findutils  12) gawk  13) gcc-libs  14) gettext  15) glibc
	16) grep 17) gzip  18) inetutils  19) iproute2  20) iputils  21) jfsutils  22) less  23) licenses
	24) linux  25) logrotate  26) lvm2  27) man-db  28) man-pages  29) mdadm  30) nano  31) netctl
	32) pacman 33) pciutils  34) pcmciautils  35) perl  36) procps-ng  37) psmisc  38) reiserfsprogs  39) s-nail
	40) sed  41) shadow  42) sysfsutils  43) systemd-sysvcompat  44) tar  45) texinfo  46) usbutils  47) util-linux
	48) vi  49) which  50) xfsprogs

*Included:*
- 1-5,7-20,22-25,27-28,30,32-37,40-47,49

*Excluded:*
- 6) dhcpcd (I'll use NetworkManager instead which has it's own DHCP client)
- 21) jfsutils
- 26) lvm2
- 29) mdadm
- 31) netctl
- 38) reiserfsprogs
- 39) s-nail
- 48) vi (I'll use neovim instead)
- 50) xfsprogs

**base-devel group:**

	1) autoconf  2) automake  3) binutils  4) bison  5) fakeroot  6) file  7) findutils
	8) flex  9) gawk  10) gcc  11) gettext  12) grep  13) groff  14) gzip  15) libtool
	16) m4  17) make  18) pacman  19) patch  20) pkg-config  21) sed  22) sudo  23) texinfo
	24) util-linux  25) which

	Enter a selection (default=all): all

## TODO

- modularize .files, improve initialization; few interesting ideas here: https://github.com/gerritwalther/dotfiles; namely:
	- git bootstrapping: email, name, ssh configs? (+ something for work repositories?)
	- .gituser/gitconfig
	- Usage section, possibility to install just part
	- backups, symlink and helper functions with logging (could be useful)
		- it could "recover" symlinked files when .files changes (something is removed) automatically
- passwords etc. to KeepassXC + figure out where to synchronize (probably still Bitbucket)
- prepare installation tool/helper/guide/wizard whatever… rewrite the installation with help of it
- something like xlunch in js+html
- create simple "nautilus" replacement in js+html
	- create "tree" view to replace nerdtree in vim
- add note about optional deps in this tutorial… make it generic (for both desktop+laptop [later maybe server])
- prepare some URL for easier installation; write just simple steps which will use it for initial installation! (+ some custom part, like pulling repositories from Bitbucket)
- prepare some "live" usb/remastered image for installation
- create some "central" place for ssh/gpg keys synchronization/distribution (make a list where I need to use them/update them/revoke them)
	- and update them regularly
- don't forget to "extract" from config file how to correctly setup dhcp + names in work network
- write chrome extension for youtube-dl (audio/video etc.)
- prepared backup+recovery plan, once a month:
	- backup all not backed up things; especially ssh+gpg keys, passwords, special settings
	- create fresh backup flash discs with backup ssh+gpg keys, passwords etc.
	- create live distro flash disc
	- reinstall all computers (according to this steps)
	- recover everything from backup!
	- update steps accordingly if something is missing!
- go through installation of all packages and check if there are other useful (optional) packages I'm missing now
- use/add config for i3lock (or some other alternative)
- other useful software:
	- file roller?, remmina, gnome-disks, gparted
	- android studio, java, apache-ant (?), virtual box (+ image win+edge/ie11), gimp, inkscape, udiskie, ranger, docker
	- libreoffice
- what to replace by my implementations:
	- xlunch
	- i3lock (with date/time)
	- KeepassXC chrome extension
	- systemd encrypt module (do not show stars while typing!)
	- polybar?
- configure fstrimmer for ssd?
- configure openvpn to Turris with NetworkManager
- add color to pacman:

	vim /etc/pacman.conf: Uncomment #Color

- setup ssh: no password, no root, timeouts etc.
- rewrite rules in i3 from title to class with usage of xprop WM_CLASS
- switch to alacritty from xfce4-terminal?
- make zsh faster (avoid usage of zplug? and implement simple replacements for current plugins? + avoid usage of nvm?)
