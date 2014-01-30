# Kiste

Notes and scripts for dev box management.


## Linux distro and hardware

These notes refer to running [Arch Linux](https://www.archlinux.org/) on the
ASUS UX301LA. Other branches of this repo track previously used setups,
for reference.


## Install and setup

This removes Windows completely and replaces it with Arch. It assumes UEFI
boot and GPT partitioning.

These notes serve as a reminder of the specific things done for my setup.
It includes links to resources that cover the process more completely and
generally.

### Windows recovery USB

Boot into Windows and create a 16GB recovery USB that can be used to restore
to factory state. The following articles are useful (and apply to ASUS as well):

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
* Create a new boot option (for the inserted Arch installer USB, booting the file `EFI/boot/loader.efiEFI/boot/loader.efi`)

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

    mkfs.ext4 /dev/md126p2
    mkfs.fat -F32 /dev/md126p1

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

    echo arch > /etc/hostname

Install packages for wireless networking:

    pacman -S iw wpa_supplicant
    pacman -S dialog

Set the root password:

    passwd

Install GRUB (blank screen issue arises with Gummiboot):

    # https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX51Vz#Installation
    # https://wiki.archlinux.org/index.php/ASUS_Zenbook_Prime_UX31A
    # https://wiki.archlinux.org/index.php/ASUS_Zenbook_UX31E
    # https://help.ubuntu.com/community/AsusZenbook

    # https://wiki.archlinux.org/index.php/Installing_with_Fake_RAID
    # http://unix.stackexchange.com/questions/71203/ubuntu-how-do-the-md-devices-get-assembled-at-bootup

    # http://wiki.gentoo.org/wiki/GRUB2#Booting_from_RAID_Array

    pacman -S grub efibootmgr

    mdadm --examine --scan >> /etc/mdadm.conf

    # TODO: CHANGE THIS TO GET EXTRA MODULES INTO MENUENTRY
    # TODO: MAKE SURE THAT THE MODULES WILL BE FOUND IN x86_64-efi/... ?
    mkdir -p /etc/grub.d
    touch /etc/grub.d/01_extra_modules
    chmod +x /etc/grub.d/01_extra_modules
    echo 'echo "insmod raid0"' >> /etc/grub.d/01_extra_modules
    echo 'echo "insmod md_mod"' >> /etc/grub.d/01_extra_modules

    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck
    grub-mkconfig -o /boot/grub/grub.cfg

    # https://wiki.archlinux.org/index.php/mkinitcpio#Using_RAID
    vi /etc/mkinitcpio.conf   # add `mdadm` to `HOOKS` and `raid0 md_mod` to `MODULES`
    mkinitcpio -p linux   # TODO: SPECIFY CORRECT KERNEL VESION WHEN BUILDING THIS !!!!!!!!!!!


    # NOTES:

    > Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
    > CPU: 0PID... Not tainted 3.12.9-1-ARCH #1

    > grub> ls
    > (hd0) (hd0,msdos2) (hd1) (hd1,gpt2) (hd1,gpt1)
    > grub> ls (hd1)
    > Device hd1: No known fielsystems detected - Sector size 512B - Total size 500113408KiB
    > grub> ls (hd1,1)
    >         Partition hd1,1: Filesystem type fat, UUID 068E-58AE - Partition start at 1024KiB - Total size 524288KiB
    > grub> ls (hd1,2)
    >         Partition hd1,2: Filesystem type ext* - Last modification time 2014-01-29 20:00:59 Wednesday, UUID 5b586628-1c53-40fb-8ed9-399ff1d4c2b2 - Partition start at 525312KiB - Total size 499588079.5KiB

Unmount partitions and reboot:

    exit   # (from chroot)
    umount -R /mnt
    reboot   # make sure install media is removed

### Post install

Set up networking to automatically connect to known WiFi.

    wifi-menu wlp2s0
    pacman -S wpa_actiond
    systemctl enable netctl-auto@wlp2s0.service

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

Activate SNA acceleration by creating `/etc/X11/xorg.conf.d/20-intel.conf` as:

    Section "Device"
            Identifier "Intel Graphics"
            Driver "intel"
            Option "AccelMethod" "sna"
    EndSection

Get KMS working, by adding the following line to `/etc/mkinitcpio.conf`:

    MODULES="i915"

and running:

    mkinitcpio -p linux
    reboot

### Install the desktop environment

Install GNOME:

    pacman -S gnome
    systemctl enable gdm.service


## Also


See the [Lenovo X1 Carbon](https://wiki.archlinux.org/index.php/Lenovo_ThinkPad_X1_Carbon)
wiki page.


## Questions

* Why doesn't my Knoppix USB boot?
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

