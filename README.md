# Arch Linux Desktop installation guide
## Goals

- minimal system based on [i3wm](http://i3wm.org/) and few utilities
- [encrypted root partition](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system)
- suspend & hibernation support for a laptop
- power management fine-tuning for a laptop

## Decisions

- use systemd-boot instead of Grub
- use dm-crypt with LUKS and Ext4
- use script & ttyrec for "recording of installation" as part of documentation
- use autologin & avoid display manager (root partition is encrypted, one password is enough)
 - but use i3lock (or similar) for manual locking and after suspend
- use arch-luks-suspend
- swap partition is used only for suspend/hibernate purposes
- use xfce4-terminal (it supports transparency, true colors, config reload on-the-fly and has less deps/is more lightweight than gnome-terminal)

## Preparation

- download the latest Arch Linux iso & sig file from https://www.archlinux.org/download/
- run `gpg2 --verify archlinux-YYYY-MM-01-dual.iso.sig archlinux-YYYY-MM-01-dual.iso`
- run `sudo imagewriter` (https://aur.archlinux.org/packages/imagewriter) and write the image to a USB flash disk

## Steps

### Wipe the disk

- boot from USB flash drive to Arch Linux

	# cryptsetup open --type plain /dev/nvme0n1 container --key-file /dev/random
	# dd if=/dev/zero of=/dev/mapper/container bs=1M status=progress
	# reboot

### Prepare partitions

- boot from USB flash drive to Arch Linux

	# gdisk /dev/sda
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

	# mkfs.fat -F32 /dev/sda1
	# cryptsetup -y -v luksFormat /dev/sda2
	# cryptsetup open /dev/sda2 cryptroot
	# mkfs.ext4 /dev/mapper/cryptroot

### Installation

	# timedatectl set-ntp true
	# wifi-menu ## setup wifi if computer is not connected on ethernet
	# mount /dev/mapper/cryptroot /mnt
	# mkdir /mnt/boot
	# mount /dev/sda1 /mnt/boot
	# pacstrap -i /mnt base base-devel ## see table below for explanation of my choices
		base> 1-5,7-20,22-25,27-28,30,32-37,40-47,49
		base-devel> all
	# genfstab -U /mnt >> /mnt/etc/fstab
	# vim /mnt/etc/fstab ## update accordingly: don't use discard option for NVME SSD drives!, use fstrim systemd timer instead
		# 
		# /etc/fstab: static file system information
		#
		# <file system>	<dir>	<type>	<options>	<dump>	<pass>
		# /dev/mapper/cryptroot
		UUID=efe2769c-f787-4fee-86e3-cb465bce4983	/         	ext4      	rw,relatime,data=ordered,errors=remount-ro	0 1

		# /dev/sda1
		UUID=73FE-9384      	/boot     	vfat      	defaults	0 1

	# arch-chroot /mnt /bin/bash
	# pacman -S zsh vim iw wpa_supplicant dialog openssh netctl dhcpcd ## to be able to continue installation after reboot! TODO uninstall later (at least netctl or dhcpcd, I don't remember)
	# vim /etc/locale.gen ## replace by sed command later
		# Uncomment following:
		cs_CZ.UTF-8 UTF-8
		en_US.UTF-8 UTF-8
	# locale-gen
	# vim /etc/locale.conf
		# Content is:
		LANG=en_US.UTF-8
	# rm /etc/localtime
	# ln -s /usr/share/zoneinfo/Europe/Prague /etc/localtime
	# hwclock --systohc --utc
	# vim /etc/mkinitcpio.conf ## (for details see: https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio)
		HOOKS="systemd autodetect modconf keyboard block sd-encrypt filesystems fsck"
	# mkinitcpio -p linux
	# mkdir /boot/EFI
	# bootctl --path=/boot install
	# vim /boot/loader/loader.conf
		# Content is:
		default arch
		timeout 0
		editor 0
	# pacman -S intel-ucode
	# cryptsetup luksUUID /dev/sda2 > /boot/loader/entries/arch.conf
	# vim /boot/loader/entries/arch.conf
			# Content is:
			title	Arch Linux
			linux	/vmlinuz-linux
			initrd	/intel-ucode.img
			initrd	/initramfs-linux.img
			options	luks.name=783a72a6-6ca9-4104-9961-11a204a5fc38=cryptroot root=/dev/mapper/cryptroot
	# vim /etc/hostname
			# Content is:
			jprokop-<some-unique-short-machine-nickname>
	# passwd ## to be able to log in as root to ssh after reboot
	# exit
	# umount -R /mnt
	# reboot

	# wifi-menu # connect to wifi
	# timedatectl set-ntp true
	# useradd -m -g users -s /usr/bin/zsh jprokop
	# gpasswd -a jprokop lp
	# gpasswd -a jprokop wheel
	# gpasswd -a jprokop network
	# gpasswd -a jprokop video
	# gpasswd -a jprokop audio
	# gpasswd -a jprokop storage
	# passwd jprokop
	# vim /etc/sudoers.d/10-jprokop
			jprokop ALL=(ALL) ALL
	# sudo -s -u jprokop # test it!
	$ sudo pacman -Sy
	$ exit
	# passwd -l root
	# reboot

- login as jprokop now:

	$ sudo wifi-menu
	$ sudo pacman -S crda wireless-regdb
	$ sudo vim /etc/conf.d/wireless-regdom
		# Uncomment following line:
		WIRELESS_REGDOM="CZ"
	$ sudo pacman -S python python-pip python2 git pkgfile
	$ sudo pacman -S xf86-video-intel mesa xorg-server xorg-xinit xf86-input-libinput
	$ sudo pacman -S i3-wm numlockx chromium xfce4-terminal
	$ sudo pkgfile --update
	$ vim ~/.xinitrc
		#!/usr/bin/env bash
		[[ -f ~/.Xresources ]] && xrdb -merge -I$HOME ~/.Xresources
		numlockx &
		exec i3

#### Included/excluded packages explanation

base group:
	 1) bash  2) bzip2  3) coreutils  4) cryptsetup  5) device-mapper  6) dhcpcd  7) diffutils  8) e2fsprogs  9) file  10) filesystem  11) findutils  12) gawk  13) gcc-libs  14) gettext  15) glibc  16) grep
	 17) gzip  18) inetutils  19) iproute2  20) iputils  21) jfsutils  22) less  23) licenses  24) linux  25) logrotate  26) lvm2  27) man-db  28) man-pages  29) mdadm  30) nano  31) netctl  32) pacman
	 33) pciutils  34) pcmciautils  35) perl  36) procps-ng  37) psmisc  38) reiserfsprogs  39) s-nail  40) sed  41) shadow  42) sysfsutils  43) systemd-sysvcompat  44) tar  45) texinfo  46) usbutils
	 47) util-linux  48) vi  49) which  50) xfsprogs
	#------------------------------
	# Excluded:
	# - 6) dhcpcd (I'll use NetworkManager instead which has it's own DHCP client)
	# - 21) jfsutils
	# - 26) lvm2
	# - 29) mdadm
	# - 31) netctl
	# - 38) reiserfsprogs
	# - 39) s-nail
	# - 48) vi (I'll use neovim instead)
	# - 50) xfsprogs
	# Included:
	# 1-5,7-20,22-25,27-28,30,32-37,40-47,49
	# I wasn't sure with:
	# - 18) inetutils seems to contain obsolete programs
	#------------------------------

base-devel group:
	 1) autoconf  2) automake  3) binutils  4) bison  5) fakeroot  6) file  7) findutils  8) flex  9) gawk  10) gcc  11) gettext  12) grep  13) groff  14) gzip  15) libtool  16) m4  17) make  18) pacman  19) patch
	 20) pkg-config  21) sed  22) sudo  23) texinfo  24) util-linux  25) which

	Enter a selection (default=all): all

# TODO

- go through my original instructions and check if there are other useful (optional) packages I'm missing now
- uninstall later at least netctl or dhcpcd, I don't remember
