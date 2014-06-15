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

* [Arch wiki page for the ASUS Zenbook UX301LA](https://wiki.archlinux.org/index.php/ASUS_UX301LA).
* [Arch wiki page for the ASUS Zenbook Prime UX31A](https://wiki.archlinux.org/index.php/ASUS_Zenbook_Prime_UX31A)

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

Enter the boot settings menu by holding `<F2>` at boot, and:

* Leave the SATA Mode Selection as RAID
* Disable Intel(R) Anti-Theft Technology
* Disable Secure Boot Control
* Create a new boot option
    (for the inserted Arch installer USB, booting the file `EFI/boot/loader.efi`)
    (this option may appear without manual creation)

Boot the Arch install USB by holding `Esc` on boot and choosing the Arch installer.

### Arch base install

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

Set up crypto for root disk:

    # backup: rsync -avxHSAX /source/ target

    cat /dev/zero > /dev/md126p2

    cryptsetup luksFormat /dev/md126p2
    cryptsetup luksDump /dev/md126p2

    # consider backup with: cryptsetup luksHeaderBackup --header-backup-file <file> <device>
    # close with: cryptsetup luksClose /dev/md126p2

    cryptsetup luksOpen /dev/md126p2 cryptroot   # appears under /dev/mapper/cryptroot

    # NOTES:
    # * useful extra info (including suspend2disk/resume): https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration
    # * suspend to RAM should work but not flush key
    #     alternative: https://github.com/vianney/arch-luks-suspend
    #     alternative: hibernate (to disk) might work better

Format the new partitions:

    mkfs.fat -F32 /dev/md126p1
    mkfs.ext4 /dev/mapper/cryptroot

Mount:

    lsblk /dev/md126
    ls -al /dev/md
    mount /dev/mapper/cryptroot /mnt
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
                              # add `encrypt` before `filesystems`, as mentioned in https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio
    mkinitcpio -p linux

    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck
    vi /etc/default/grub      # add `cryptdevice=/dev/sdaX:cryptroot`, as mentioned in https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_the_boot_loader
    grub-mkconfig -o /boot/grub/grub.cfg

Unmount partitions and reboot:

    exit   # (from chroot)
    umount -R /mnt
    cryptsetup luksClose /dev/mapper/cryptroot
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

Enable [TLP](https://wiki.archlinux.org/index.php/TLP) for power managment:

    pacman -S tlp
    systemctl enable tlp
    systemctl enable tlp-sleep

Fix brightness [function keys](https://wiki.archlinux.org/index.php/ASUS_UX301LA#Function_Keys):

    vi /etc/default/grub   # set `GRUB_CMDLINE_LINUX_DEFAULT="quiet acpi_osi="`
    grub-mkconfig -o /boot/grub/grub.cfg

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

Enable `multlib` by uncommenting the `[multlib]` section in `/etc/pacman.conf`
and updating the package list with `pacman -Syyu` (will also do a system upgrade).

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
    sudo pacman -S strace
    sudo pacman -S xclip
    sudo pacman -S traceroute
    sudo pacman -S subversion
    sudo pacman -S stow
    sudo pacman -S wget
    sudo pacman -S curl
    sudo pacman -S firefox
    sudo pacman -S flashplugin
    sudo pacman -S chromium
    sudo pacman -S opera
    sudo pacman -S calibre
    pacaur -S qbittorrent
    sudo pacman -S vlc
      sudo pacman -S ffmpegthumbnailer gstreamer0.10-ffmpeg
      sudo pacman -S gstreamer0.10-base-plugins gstreamer0.10-good
    sudo pacman -S extundelete
    sudo pacman -S ext4magic
    sudo pacman -S gnumeric
    sudo pacman -S liferea
    sudo pacman -S xchm
    sudo pacman -S python-pip
    sudo pacman -S python2-pip
    sudo pacman -S jdk7-openjdk
      sudo pacman -S apache-ant
    sudo pacman -S arandr
    sudo pacman -S s3cmd
    sudo pacman -S sox
    sudo pacman -S rhythmbox
      sudo pacman -S gst-plugins-ugly
      sudo pacman -S gst-plugins-bad
      sudo pacman -S gst-libav
      sudo pacman -S libdmapsharing
      sudo pacman -S brasero
    sudo pacman -S ack
    sudo pacman -S gdal
    sudo pacman -S fatrat
    sudo pacman -S gparted
      sudo pacman -S dosfstools
      sudo pacman -S jfsutils
      sudo pacman -S f2fs-tools
      sudo pacman -S btrfs-progs
      sudo pacman -S exfat-utils
      sudo pacman -S ntfs-3g
      sudo pacman -S reiserfsprogs
      sudo pacman -S xfsprogs
      sudo pacman -S nilfs-utils
      sudo pacman -S polkit
      sudo pacman -S gpart
      sudo pacman -S mtools
    sudo pacman -S kdeutils-filelight
    sudo pacman -S unrar
    sudo pacman -S phantomjs
    sudo pacman -S elinks
    sudo pacman -S ghc
      sudo pacman -S cabal-install
      sudo pacman -S haddock
      sudo pacman -S happy
      sudo pacman -S alex
    sudo pacman -S libreoffice
      sudo pacman -S libreoffice-gnome
      sudo pacman -S hunspell
      sudo pacman -S hunspell-en
    sudo pacman -S wireshark-gtk
    sudo pacman -S sqlitebrowser
    sudo pacman -S i3-wm dmenu i3lock i3status
    sudo pacman -S go
    sudo pacman -S iotop
    sudo pacman -S keepass
    sudo pacman -S docker && sudo systemctl enable docker
    sudo pacman -S qalculate-gtk
    sudo pacman -S gimp
    sudo pacman -S kdesdk-okteta
    sudo pacman -S gnu-netcat
    pacaur -S brackets-bin
    pacaur -S mobac
    sudo pacman -S expect  # for unbuffer
    sudo pacman -S gptfdisk
    sudo pacman -S rsync
    sudo pacman -S redis
    sudo pacman -S youtube-dl
    pacaur -S coursera-dl-git
    pacaur -S bmon
    pacaur -S sysdig
    sudo pacman -S moreutils
    sudo pacman -S nfs-utils

[Install from the AUR](https://wiki.archlinux.org/index.php/AUR#Installing_packages) the AUR tools:

* [`package-query`](https://aur.archlinux.org/packages/package-query/)
* [`yaourt`](https://aur.archlinux.org/packages/yaourt/)
* [`pacaur`](https://aur.archlinux.org/packages/pacaur/)

Install extra AUR packages:

    yaourt -Sa google-chrome
      yaourt -Sa ttf-google-fonts-git
    yaourt -Sa hipchat
    yaourt -Sa aerofs
    yaourt -Sa briss
    yaourt -Sa sublime-text
    yaourt -Sa ledger
    yaourt -Sa dropbox
      yaourt -Sa nautilus-dropbox
    yaourt -Sa ansible
    yaourt -Sa zeal-git
    yaourt -Sa shutter
    yaourt -Sa qgis-git   # this version without grass
    yaourt -Sa jq
    yaourt -Sa v8
    pacaur -S randomsound

Let `wheel` users run wireshark in its group (`sudo -g wireshark wireshark`):

    echo '%wheel ALL=(:wireshark) /usr/bin/wireshark, /usr/bin/tshark' | sudo tee /etc/sudoers.d/wireshark

Set up default applications for Gnome (e.g. so filelight doesn't steal responsiblity for `inode/directory`):

    pacaur -S gnome-defaults-list
    cp /etc/gnome/defaults.list ~/.local/share/applications/defaults.list

Install and setup [VirtualBox](https://wiki.archlinux.org/index.php/VirtualBox):

    sudo pacman -S virtualbox
    sudo pacman -S virtualbox-host-modules
    sudo pacman -S qt4
    sudo pacman -S net-tools
    sudo pacman -S virtualbox-guest-iso
    sudo depmod -a   # update the kernel dependency modules database
    sudo modprobe vboxdrv
    echo vboxdrv | sudo tee -a /etc/modules-load.d/virtualbox.conf
    echo vboxnetadp | sudo tee -a /etc/modules-load.d/virtualbox.conf
    echo vboxnetflt | sudo tee -a /etc/modules-load.d/virtualbox.conf
    echo vboxpci | sudo tee -a /etc/modules-load.d/virtualbox.conf
    sudo gpasswd -a $USER vboxusers
    virtualbox &
    sudo pacman -S vagrant

Install and setup MariaDB (mysqld):

    sudo pacman -S mariadb
    sudo systemctl enable mysqld.service
    sudo systemctl start mysqld.service
    mysql_secure_installation
    sudo systemctl restart mysqld.service
    pacaur -S mysql-workbench

Install and setup [PostgreSQL](https://wiki.archlinux.org/index.php/PostgreSQL):

    sudo pacman -S postgresql
    sudo pacman -S pgadmin3
    sudo su - postgres -c "initdb --locale en_AU.UTF-8 -E UTF8 -D '/var/lib/postgres/data'"
    sudo systemctl start postgresql
    sudo systemctl enable postgresql

    sudo su - postgres
    createuser --interactive   # role: chris, superuser: y
    createdb chris
    exit

Install and setup MongoDB:

    sudo pacman -S mongodb
    sudo systemctl start mongodb

Get Windows product key from the BIOS data:

    sudo pacman -S iasl
    mkdir acpi && cd acpi
    sudo acpidump > acpi.dat
    acpixtract -a acpi.dat
    hexdump -C msdm.dat   # key is the last XXXXX-XXXXX-XXXXX-XXXXX-XXXXX

Install `bitcoind` (see [the wiki](https://wiki.archlinux.org/index.php/Bitcoin#Installation)):

    sudo pacman -S bitcoin-daemon
    sudo useradd -m bitcoin
    sudo cp /usr/share/doc/bitcoin-daemon/examples/bitcoind.conf /home/bitcoin/.bitcoin/bitcoin.conf
    sudo chown bitcoin:bitcoin /home/bitcoin/.bitcoin/bitcoin.conf
    sudo chmod 600 /home/bitcoin/.bitcoin/bitcoin.conf
    vim /etc/systemd/system/bitcoind.service   # see below
    sudo systemctl start bitcoind
    sudo systemctl status bitcoind   # take recommended password and put it in config
    sudo vim /home/bitcoin/.bitcoin/bitcoin.conf
    sudo systemctl restart bitcoind

`bitcoind` service definition:

    [Unit]
    Description=Bitcoin daemon service
    After=network.target

    [Service]
    Type=simple
    User=bitcoin
    ExecStart=/usr/bin/bitcoind

    [Install]
    WantedBy=multi-user.target

Other Bitcoin and crypto stuff:

    pacaur -S multibit
    pacaur -S electrum python2-zbar
    pacaur -S sx-git
    pacman -S tor vidalia
    pacaur -S pybitmessage

Android file access:

    sudo pacman -S gvfs gvfs-mtp

Android dev env:

    pacaur -S android-sdk
    pacaur -S android-sdk-platform-tools
    pacaur -S android-sdk-build-tools
    sudo android
    sudo chmod -R 755 /opt/android-sdk

    pacaur -S android-studio

Printer/scanner setup:

    sudo pacman -S cups ghostscript gsfonts avahi cups-browsed
    pacaur -S epson-inkjet-printer-stylus-office-tx610fw-series
    sudo systemctl start avahi-daemon.service
    sudo systemctl enable avahi-daemon.service
    sudo systemctl start cups.service
    sudo systemctl enable cups.service
    sudo pacman -S system-config-printer
    # then run GUI app "Print Settings"

    pacaur -S iscan iscan-plugin-network
    sudo vim /etc/sane.d/epkowa.conf  # add line 'net {IP_OF_SCANNER}'
    # then run GUI app "Image Scan! for Linux"

OCR (see [guide on the wiki](https://wiki.archlinux.org/index.php/List_of_applications/Documents#OCR_software)):

    sudo pacman -S tesseract tesseract-data-eng
    pacaur -S gscan2pdf
    pacaur -S pdftk

Google Earth

    pacaur -S google-earth
    sudo pacman -S lib32-intel-dri

Adjust DPI settings in `dconf-editor` under `/org/gnome/desktop/interface/text-scaling-factor` and `scaling`.
Or, use `xrandr`'s `--scale` or `--transform` options.

Install user files and tools, into home directory, including:

* Ruby (via `rbenv`)
* NodeJS (via `nvm`)
* `autoenv`
* Powerline

Add extra startup applications (e.g. Skype), by pressing `<Alt>-<F2>` and entering `gnome-session-properties`.

Remap CAPS as CTRL: in `/etc/X11/xorg.conf.d/10-evdev.conf`, in the keyboard section, add:

    Option "XkbOptions" "ctrl:nocaps"

The wiki page [Keyboard configuration in Xorg](https://wiki.archlinux.org/index.php/Keyboard_configuration_in_Xorg) may also be useful.

### Next

* MTU permanent setting

* windows

* postgis & template DBs

* home dir encryption

* sort Old data
* crashplan

* read [Arch wiki page for the ASUS Zenbook Prime UX31A](https://wiki.archlinux.org/index.php/ASUS_Zenbook_Prime_UX31A)
* https://wiki.archlinux.org/index.php/General_Recommendations
* https://wiki.archlinux.org/index.php/Laptop
* https://wiki.archlinux.org/index.php/lm_sensors, https://wiki.archlinux.org/index.php/Fan_Speed_Control


## Issues

* google chrome doesn't do chinese text

* Keyboard settings in `/etc/X11/xorg.conf.d/10-evdev.conf` aren't working.

* Login script to turn off keyboard backlight doesn't seem to run on login (or gets reversed after running).

* Switching users and returning kills original login session (since ~2014-04-18).

* Copying into system clipboard (e.g. in vim) doesn't copy to tmux clipboard

* Copying multiple lines in tmux generates random text in adjacent panes.

* Nexus 5 audio access remains a problem. I tried:
  * [adding a udev rule](https://wiki.archlinux.org/index.php/MTP#Using_media_players) -
    This didn't help, and is probably unnecessary because it is covered by the default rules in `/usr/lib/udev/rules.d/69-libmtp.rules`.
  * [creating a `.is_audio_player` file](http://almost-a-technocrat.blogspot.com.au/2010/11/isaudioplayer.html) -
    This made Rhythmbox show existing music, but writing music to the device failed with
    `Unable to send file to MTP device: PTP Layer error 02ff: get_storage_freespace(): could not get storage info.`
  The workaround is to do manual music management via the filesystem.

* High DPI and mixed DPI setups not handled well. See:
  * http://blogs.gnome.org/alexl/2013/06/28/hidpi-support-in-gnome/
  * http://vincent.jousse.org/tech/archlinux-retina-hidpi-macbookpro-xmonad/

## Questions

* Windows reinstall issue: "This product key cannot be used to install a retail version of Windows 8"


## TODO

* add skype with some lockdown
* remove hipchat
* IRC:
    * Quassel
* MS fonts? https://wiki.archlinux.org/index.php/MS_Fonts
* ntp or that other one?
* cron alternative for intermittent processing/network
* emacs
* Heroku toolbelt: [https://toolbelt.herokuapp.com/debian]()
* Make PKGBUILDs of:
  - [prax](https://github.com/ysbaddaden/prax) (needs systemd service)
  - [pdfocr](https://github.com/gkovacs/pdfocr/)
* Clipboard manager: [http://parcellite.sourceforge.net/?page_id=2]()
* Check out [XFCE recommended apps](http://wiki.xfce.org/recommendedapps)

