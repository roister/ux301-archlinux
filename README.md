# Kiste

Notes and scripts for dev box management.


## Linux distro and hardware

These notes refer to running [Arch Linux](https://www.archlinux.org/) on the
[Lenovo X1 Carbon](http://en.wikipedia.org/wiki/ThinkPad_X1_Carbon).
Other branches of this repo track previously used setups, for reference.


## Install and setup

This removes Windows completely and replaces it with Arch. It assumes UEFI
boot and GPT partitioning.

These notes serve as a reminder of the specific things done for my setup.
It includes links to resources that cover the process more completely and
generally.

### Windows recovery USB

Boot into Windows and create a 16GB recovery USB that can be used to restore
to factory state. The following articles are useful:

* [Understanding hard drive partitions on Lenovo systems with Microsoft Windows 7 and Windows 8](http://support.lenovo.com/en_US/detail.page?DocID=HT077144)
* [Methodology to create Recovery Media and reload a Lenovo Think system with Microsoft Windows 8 preload](http://support.lenovo.com/en_US/downloads/detail.page?DocID=HT076024)

### Arch installation USB

Following the [USB Installation Media](https://wiki.archlinux.org/index.php/USB_Installation_Media)
guide, download the [latest install ISO](https://www.archlinux.org/download/)
and write it onto a USB stick with:

    dd bs=4M if=/path/to/archlinux.iso of=/dev/sdX && sync

Enter the boot settings menu by holding `Enter` at boot, and:

* Set boot mode to UEFI only
* Disable secure boot
* Activate VT (recommended by [wiki](https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_X1_Carbon#KMS))

Boot the Arch install USB by holding `F12` on boot.

### Arch base intall

Following the [Arch Beginner's Guide](https://wiki.archlinux.org/index.php/Beginners'_Guide),
perform the following steps for the base install:

Confirm you're booted into UEFI mode by checking that EFI variables are available:

    efivar -l

Prepare the disk (with GPT partitions), using:

    lsblk
    cgdisk /dev/sda   # or gdisk, which shows a partition table scan on start

...delete all partitions and create two new partitions:

* [EFI system partition](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#GPT_partitioned_disks)
  of 512MiB (code `EF00` or `ef00`), as `/dev/sda1`
* Linux partition for Arch using the rest, as `/dev/sda2`

Format the new partitions:

    mkfs.ext4 /dev/sda2
    mkfs.fat -F32 /dev/sda1

Mount:

    lsblk /dev/sda
    mount /dev/sda2 /mnt
    mkdir -p /mnt/boot
    mount /dev/sda1 /mnt/boot

Establish a WiFi Internet connection:

    iw dev
    ip link set wlp3s0 up
    ip link show wlp3s0
    wifi-menu wlp3s0
    ping google.com

Move a preferable mirror to the top of the list:

    vi /etc/pacman.d/mirrorlist

Install the base system:

    pacstrap -i /mnt base

Generate an `fstab` file:

    genfstab -U -p /mnt >> /mnt/etc/fstab
    vi /mnt/etc/fstab   # just to check

Chroot to base system:

    arch-chroot /mnt /bin/bash

Set the locale (`en_AU.UTF-8 UTF-8`):

    vi /etc/locale.gen
    locale-gen
    echo LANG=en_AU.UTF-8 > /etc/locale.conf
    export LANG=en_AU.UTF-8
    locale   # to check available vars

Set font with support for more languages:

    vi /etc/vconsole.conf   # and add line `FONT=Lat2-Terminus16`
    setfont Lat2-Terminus16

Set localtime link to appropriate zone file:

    ln -s /usr/share/zoneinfo/Australia/Melbourne /etc/localtime

Set the hardware clock to UTC:

    hwclock --systohc --utc

Set the hostname:

    echo arch > /etc/hostname

Install packages for wireless networking:

    pacman -S iw wpa_supplicant
    pacman -S dialog

Set the root password:

    passwd

Install GRUB (blank screen issue arises with Gummiboot):

    pacman -S grub efibootmgr
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck
    grub-mkconfig -o /boot/grub/grub.cfg

Unmount partitions and reboot:

    exit   # (from chroot)
    umount -R /mnt
    reboot   # make sure install media is removed

### Post install

Set up networking to automatically connect to known WiFi

## Also

See about KMS, VT


## Questions

* My install of Gummiboot gets the screen blanking issue, but the install ISO seems to
  use it without problems. Why does the ISO one work?

## TODO

* ntp?
* passwords:
  * [http://keepass.info]()
  * [http://blog.sanctum.geek.nz/linux-crypto-passwords/]()
* feed reader:
  * [http://lifehacker.com/5992404/how-to-build-your-own-syncing-rss-reader-with-tiny-tiny-rss-and-kick-google-reader-to-the-curb]()
  * desktop: [http://lzone.de/liferea/]()
  * online for sync: [http://tt-rss.org/redmine/projects/tt-rss/wiki]()
  * mobile: [http://tt-rss.org/redmine/projects/tt-rss-android/wiki]()
* [http://www.omgubuntu.co.uk/2012/05/beatbox-music-player-sees-new-release-on-ubuntu]()
* Heroku toolbelt: [https://toolbelt.herokuapp.com/debian]()
* Alternative to pow rack dev server: [https://github.com/ysbaddaden/prax]()
* [http://askubuntu.com/questions/68809/how-to-format-a-usb-or-external-drive]()
* Clipboard manager: [http://parcellite.sourceforge.net/?page_id=2]()
* [https://apps.ubuntu.com/cat/applications/d4x/]()
* Check out [http://wiki.xfce.org/recommendedapps]()

