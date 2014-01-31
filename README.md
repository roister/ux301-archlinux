# Kiste

Notes and scripts for dev box management.


## Linux distro and hardware

These notes refer to running [Arch Linux](https://www.archlinux.org/) on the ASUS UX301LA.
Other branches of this repo track previously used setups, for reference.


## Install and setup

This removes Windows completely and replaces it with Arch. It assumes UEFI
boot and GPT partitioning.

These notes serve as a reminder of the specific things done for my setup.
It includes links to resources that cover the process more completely and
generally.

### Resources

* [Arch wiki page for the ASUS UX301LA](https://wiki.archlinux.org/index.php/ASUS_UX301LA).

### Windows recovery USB

Boot into Windows and create a 16GB recovery USB that can be used to restore
to factory state. The following articles are useful (and do apply ASUS systems):

* [Understanding hard drive partitions on Lenovo systems with Microsoft Windows 7 and Windows 8](http://support.lenovo.com/en_US/detail.page?DocID=HT077144)
* [Methodology to create Recovery Media and reload a Lenovo Think system with Microsoft Windows 8 preload](http://support.lenovo.com/en_US/downloads/detail.page?DocID=HT076024)

### Arch installation USB

Following the [USB Installation Media](https://wiki.archlinux.org/index.php/USB_Installation_Media)
guide, download the [latest install ISO](https://www.archlinux.org/download/)
and write it onto a USB stick with:

    dd bs=4M if=/path/to/archlinux.iso of=/dev/sdX && sync

Enter the boot settings menu by holding `F2` at boot, and:

* Leave the SATA Mode Selection as RAID
* Disable Intel(R) Anti-Theft Technology
* Disable Secure Boot Control
* Create a new boot option
    (for the inserted Arch installer USB, booting the file `EFI/boot/loader.efi`)
    (this option may appear without manual creation)

Boot the Arch install USB by holding `Esc` on boot and choosing the Arch installer.

### Arch base intall

Following the [Arch Beginner's Guide](https://wiki.archlinux.org/index.php/Beginners'_Guide),
perform the following steps for the base install:

Confirm you're booted into UEFI mode by checking that EFI variables are available:

    efivar -l

Inspect the current disks, partitions and RAID setup (more RAID info [here](https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz#Installation)):

    lsblk
    mdadm -D /dev/md126
    mdadm -E /dev/md126

Prepare the disk (with GPT partitions). Using:

    cgdisk /dev/md126   # or gdisk, which shows a partition table scan on start

...delete all partitions and create two new partitions:

* [EFI system partition](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#GPT_partitioned_disks)
  of 512M (code `EF00` or `ef00`)
* Linux partition for Arch using the rest

Format the new partitions:

    mkfs.fat -F32 /dev/md126p1
    mkfs.ext4 /dev/md126p2

Mount:

    lsblk /dev/md126
    ls -al /dev/md
    mount /dev/md126p2 /mnt
    mkdir -p /mnt/boot
    mount /dev/md126p1 /mnt/boot

Establish a WiFi Internet connection:

    iw dev
    ip link set wlp2s0 up
    ip link show wlp2s0
    wifi-menu wlp2s0
    ping google.com

Move a preferable mirror to the top of the list:

    vi /etc/pacman.d/mirrorlist

Install the base system:

    pacstrap -i /mnt base

Generate an `fstab` file:

    genfstab -U -p /mnt >> /mnt/etc/fstab
    vi /mnt/etc/fstab   # Set last param of EFI partion to 0 (don't check)

Chroot to base system:

    arch-chroot /mnt /bin/bash

Set the locale (`en_AU.UTF-8 UTF-8`):

    vi /etc/locale.gen
    locale-gen
    echo LANG=en_AU.UTF-8 > /etc/locale.conf
    export LANG=en_AU.UTF-8
    locale   # to check available vars

Set localtime link to appropriate zone file:

    ln -s /usr/share/zoneinfo/Australia/Melbourne /etc/localtime

Set the hardware clock to UTC:

    hwclock --systohc --utc

Set the hostname:

    echo zenbook > /etc/hostname

Install packages for wireless networking:

    pacman -S iw wpa_supplicant dialog

Set the root password:

    passwd

Install GRUB:

    pacman -S grub efibootmgr

    mdadm --examine --scan >> /etc/mdadm.conf   # may not be necessary

    vi /etc/mkinitcpio.conf   # add `mdadm_udev` to `HOOKS`, as mentioned in https://wiki.archlinux.org/index.php/ASUS_UX301LA
    mkinitcpio -p linux

    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck
    grub-mkconfig -o /boot/grub/grub.cfg

Unmount partitions and reboot:

    exit   # (from chroot)
    umount -R /mnt
    reboot   # make sure install media is removed

### Post install

Reconnect to the Internet:

    wifi-menu wlp2s0

Sudo:

    pacman -S sudo

Create group and user, allow to sudo:

    groupadd chris
    useradd -m -g chris -G chris,wheel -s /bin/bash chris
    chfn chris
    passwd chris
    vi /etc/sudoers   # allow users in wheel group to sudo

Sound:

    pacman -S alsa-utils

X Windows, 3D and video driver, touchpad:

    pacman -S xorg-server xorg-server-utils xorg-xinit
    pacman -S mesa
    pacman -S xf86-video-intel
    pacman -S xf86-input-synaptics

### Install the desktop environment

Install GNOME:

    pacman -S gnome
    systemctl enable gdm.service

Enable network manager:

    systemctl enable NetworkManager.service

Do [tearing fix](https://wiki.archlinux.org/index.php/GNOME#Tear-free_video_with_Intel_HD_Graphics):

    echo 'CLUTTER_PAINT=disable-clipped-redraws:disable-culling' >> /etc/environment
    restart

### More stuff

Install extra packages:

    sudo pacman -S base-devel
    sudo pacman -S openssh
    sudo pacman -S openssl
    sudo pacman -S git
    sudo pacman -S tk
    sudo pacman -S tcl
    sudo pacman -S ruby
    sudo pacman -S gvim
    sudo pacman -S tmux
    sudo pacman -S xclip
    sudo pacman -S traceroute
    sudo pacman -S subversion
    sudo pacman -S stow
    sudo pacman -S wget
    sudo pacman -S curl
    sudo pacman -S firefox
    sudo pacman -S chromium
    sudo pacman -S calibre

Adjust DPI settings in `dconf-editor` under `/org/gnome/desktop/interface/text-scaling-factor` and `scaling`.

[Install from the AUR](https://wiki.archlinux.org/index.php/AUR#Installing_packages) the AUR tools:

* [`package-query`](https://aur.archlinux.org/packages/package-query/)
* [`yaourt`](https://aur.archlinux.org/packages/yaourt/)
* [`pacaur`](https://aur.archlinux.org/packages/pacaur/)

Install extra AUR packages:

    sudo yaourt -Sa google-chrome
    sudo yaourt -Sa ttf-google-fonts-git
    sudo yaourt -Sa hipchat

Install and setup [PostgreSQL](https://wiki.archlinux.org/index.php/PostgreSQL):

    sudo pacman -S postgresql
    sudo su - postgres -c "initdb --locale en_AU.UTF-8 -E UTF8 -D '/var/lib/postgres/data'"
    sudo systemctl start postgresql
    sudo systemctl enable postgresql

    sudo su - postgres
    createuser --interactive   # role: chris, superuser: y
    createdb chris
    exit

[Install](https://extensions.gnome.org/) GNOME extensions and [configure](https://extensions.gnome.org/local/) them:

* [Put Windows](https://extensions.gnome.org/extension/39/put-windows/)


### Next

Revisit the wiki page.


## Questions

* Why doesn't my Knoppix USB boot?


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

