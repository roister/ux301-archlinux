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
to factory state. The following articles are useful (and do apply to ASUS
systems):

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

    # cat /dev/zero > /dev/md126p2

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
    wifi-menu wlp2s0
    ping google.com

Move a preferable mirror to the top of the list:

    vi /etc/pacman.d/mirrorlist

Install the base system:

    pacstrap -i /mnt base base-devel
    swapon /dev/sda3 /dev/sdb3

Generate an `fstab` file:

    genfstab -U -p /mnt >> /mnt/etc/fstab
    vi /mnt/etc/fstab   # Set last param of EFI partion to 0 (don't check)

Optimise for SSD:

    vi /mnt/etc/lvm/lvm.conf     # :s/issue_discards = 0/issue_discards = 1/
    # Add kernel parameter: elevator=noop (maybe not...)
	
	# Change from nfq scheduler to noop. 
	# Create: /mnt/etc/udev/rules.d/60-schedulers.rules
    	# set deadline scheduler for non-rotating disks
    	ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="deadline"
	 
	# Reduce swappiness (thus less SSD wear)
	# Create: /mnt/etc/sysctl.d/99-sysctl.conf
		vm.swappiness=10
	#
	#

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

Grab some useful stuff

    pacman -S iotop htop

NTP time synchronization with [Chrony](https://wiki.archlinux.org/index.php/Chrony):

    pacman -S chrony
    sudo systemctl start chrony.service
    sudo systemctl enable chrony.service

Set the hostname:

    echo myhostname > /etc/hostname

And in `/etc/hosts`:

    127.0.0.1 .... localhost myhostname
    ::1       .... localhost myhostname

Install packages for wireless networking:

    pacman -S iw wpa_supplicant dialog

Set the root password:

    passwd

Install rEFInd:

    pacman -Ss refind-efi
	vi /etc/mkinitcpio.conf     # add lvm2 AFTER block and BEFORE filesystems
	mkinitcpio -p linux
	refind-install


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

CPU microcode updates (need to tell Grub to load the initrd as described [on the wiki](https://wiki.archlinux.org/index.php/Microcode#Enabling_Intel_Microcode_Updates)):

    pacman -S intel-ucode

Sound:

    pacman -S alsa-utils

X Windows, 3D and video driver, touchpad, hardware accelerated video decoding:

    pacman -S xorg-server xorg-server-utils xorg-xinit
    pacman -S mesa
    pacman -S xf86-video-intel
    pacman -S xf86-input-synaptics
    pacman -S libva-intel-driver

    # establish custom config dir (some of this will later be modified)
    cd /etc/X11/xorg.conf.d/
    ls -l 10-evdev.conf || cp /usr/share/X11/xorg.conf.d/10-evdev.conf .
    ls -l 50-synaptics.conf || cp /usr/share/X11/xorg.conf.d/50-synaptics.conf .

Enable [TLP](https://wiki.archlinux.org/index.php/TLP) for power managment:

    pacman -S tlp
    systemctl enable tlp
    systemctl enable tlp-sleep

Fix brightness [function keys](https://wiki.archlinux.org/index.php/ASUS_UX301LA#Function_Keys):

    vi /etc/default/grub   # set `GRUB_CMDLINE_LINUX_DEFAULT="quiet acpi_osi="`
    grub-mkconfig -o /boot/grub/grub.cfg

Extra entropy from CPU timings:

    pacman -S haveged
    systemctl start haveged
    systemctl enable haveged

### Install the desktop environment

Install GNOME:

    pacman -S gnome
    pacman -S gnome-tweak-tool
    systemctl enable gdm.service

For Bluetooth:

    pacman -S bluez-libs bluez-utils bluez-firmware

For PulseAudio preference editing:

    pacman -S paprefs
    pacman -S pavucontrol

Enable network manager:

    systemctl enable NetworkManager.service

Lower WiFi MTU (to avoid [path MTU black holes](http://www.fir3net.com/Terms-and-Concepts/path-mtu-path-mtu-black-holes.html) experienced with some websites):

    vim /etc/NetworkManager/dispatcher.d/20-mtu
    chown root:root /etc/NetworkManager/dispatcher.d/20-mtu
    chmod 755 /etc/NetworkManager/dispatcher.d/20-mtu


    #!/bin/bash

    INTERFACE=$1
    STATE=$2

    if [ "$STATE" = "up" ] && [ "$INTERFACE" = "wlp2s0" ]; then
        ifconfig "$INTERFACE" mtu 1492
    fi

Do the [xorg Intel tearing fix](https://wiki.archlinux.org/index.php/Intel_Graphics#Tear-free_video):

    # /etc/X11/xorg.conf.d/20-intel.conf

    Section "Device"
       Identifier  "Intel Graphics"
       Driver      "intel"
       Option      "TearFree"    "true"
    EndSection

There is a [GNOME tearing fix](https://wiki.archlinux.org/index.php/GNOME#Tear-free_video_with_Intel_HD_Graphics),
but the xorg one seems to work better (and works outside of GNOME).


### More stuff

Enable `multlib` by uncommenting the `[multlib]` section in `/etc/pacman.conf`
and updating the package list with `pacman -Syyu` (will also do a system upgrade).

Then, for 32-bit 3D video acceleration:

    sudo pacman -S lib32-intel-dri

Install extra packages:

    sudo pacman -S base-devel
    sudo pacman -S openssh
    sudo pacman -S openssl
    sudo pacman -S bash-completion
    sudo pacman -S git tk tcl gsfonts
    sudo pacman -S bzr
    sudo pacman -S ruby
    sudo pacman -S gvim
    sudo pacman -S screen
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
    sudo pacman -S espeak
    sudo pacman -S ack
    sudo pacman -S gdal
    sudo pacman -S fatrat
    sudo pacman -S gparted
      sudo pacman -S dosfstools
      sudo pacman -S jfsutils
      sudo pacman -S f2fs-tools
      sudo pacman -S btrfs-progs
      sudo pacman -S exfat-utils && pacaur -Sy exfat-git && sudo modprobe exfat
      sudo pacman -S ntfs-3g
      sudo pacman -S reiserfsprogs
      sudo pacman -S xfsprogs
      sudo pacman -S nilfs-utils
      sudo pacman -S polkit
      sudo pacman -S gpart
      sudo pacman -S mtools
    sudo pacman -S filelight
    sudo pacman -S unrar
    pacaur -Sy rar
    sudo pacman -S phantomjs
    sudo pacman -S elinks
    sudo pacman -S ghc
      sudo pacman -S cabal-install
      sudo pacman -S haddock
      sudo pacman -S happy
      sudo pacman -S alex
    sudo pacman -S libreoffice-fresh
      sudo pacman -S libreoffice-fresh-en-GB
      sudo pacman -S libreoffice-fresh-de
      sudo pacman -S libreoffice-fresh-fr
      sudo pacman -S libreoffice-fresh-ru
      sudo pacman -S libreoffice-fresh-zh-CN
      sudo pacman -S hunspell
      sudo pacman -S hunspell-en
      sudo pacman -S hunspell-de
    sudo pacman -S wireshark-gtk
    sudo pacman -S sqlitebrowser
    sudo pacman -S i3-wm dmenu i3lock i3status
    sudo pacman -S go
    sudo pacman -S iotop iftop
    sudo pacman -S sysstat
    sudo pacman -S keepass
    sudo pacman -S docker lxc lua-filesystem lua-alt-getopt && sudo systemctl enable docker && gpasswd -a $USER docker
    sudo pacman -S qalculate-gtk
    sudo pacman -S gimp
    sudo pacman -S kdesdk-okteta # hex editor
    sudo pacman -S gnu-netcat
    sudo pacman -S time
    sudo pacman -S pv
    sudo pacman -S parallel
    pacaur -Sy entr
    pacaur -Sy brackets-bin
    pacaur -Sy mobac
    sudo pacman -S expect  # for unbuffer
    sudo pacman -S gptfdisk
    sudo pacman -S rsync
    sudo pacman -S redis
      pacaur -Sy redis-desktop-manager
    sudo pacman -S youtube-dl
    pacaur -Sy coursera-dl-git
    pacaur -Sy bmon
    pacaur -Sy sysdig
    sudo pacman -S moreutils
    sudo pacman -S nfs-utils
    sudo pacman -S inkscape uniconverter
    sudo pacman -S imagemagick
    sudo pacman -S zsh
    sudo pacman -S dnsutils
    sudo pacman -S file-roller # GUI archive manager for GNOME

[Install from the AUR](https://wiki.archlinux.org/index.php/AUR#Installing_packages) the AUR tools:

TODO: AUR helpers are needed earlier on

* [`package-query`](https://aur.archlinux.org/packages/package-query/)
* [`pacaur`](https://aur.archlinux.org/packages/pacaur/)

Install extra AUR packages:

    pacaur -Sy google-chrome
      pacaur -Sy google-talkplugin

    # good free fonts
    sudo pacman -S ttf-liberation
    pacaur -Sy ttf-google-fonts-git
    sudo pacman -S ttf-linux-libertine
    sudo pacman -S ttf-freefont
    # CJKV
    sudo pacman -S opendesktop-fonts
    sudo pacman -S wqy-microhei
    sudo pacman -S wqy-zenhei
    sudo pacman -S ttf-arphic-ukai
    sudo pacman -S ttf-arphic-uming
    pacaur -Sy ttf-tw
    pacaur -Sy ttf-mplus
    sudo pacman -S ttf-baekmuk
    # Apple, non-free
    pacaur -Sy ttf-mac-fonts

    pacaur -Sy briss
    pacaur -Sy sublime-text
    pacaur -Sy atom-editor
    pacaur -Sy dropbox
      pacaur -Sy nautilus-dropbox
    pacaur -Sy ansible
    pacaur -Sy zeal-git
    pacaur -Sy shutter
    pacaur -Sy qgis-git   # this version without grass
    pacaur -Sy jq
    pacaur -Sy v8
    pacaur -Sy randomsound
    pacaur -Sy lastpass

Let `wheel` users run wireshark in its group (`sudo -g wireshark wireshark`):

    echo '%wheel ALL=(:wireshark) /usr/bin/wireshark, /usr/bin/tshark' | sudo tee /etc/sudoers.d/wireshark

Set up default applications for Gnome:

    pacaur -Sy gnome-defaults-list
    cp /etc/gnome/defaults.list ~/.local/share/applications/defaults.list

Take responsiblity for `inode/directory` away from filelight:

    sudo vim /usr/share/applications/org.kde.filelight.desktop  # comment out MimeType line
    sudo vim /usr/share/kservices5/filelightpart.desktop        # comment out MimeType line
    sudo update-desktop-database

Music

    sudo pacman -S gstreamer
      sudo pacman -S gst-libav
      sudo pacman -S gst-plugins-base
      sudo pacman -S gst-plugins-good
      sudo pacman -S gst-plugins-bad
      sudo pacman -S gst-plugins-ugly
      sudo pacman -S gst-vaapi

    sudo pacman -S rhythmbox
      sudo pacman -S libdmapsharing
      sudo pacman -S brasero

    sudo pacman -S puddletag

    sudo pacman -S beets
      sudo pip2 install requests # for fetchart plugin
      sudo pip2 install python-itunes # for fetchart plugin source
      sudo pip2 install flask # for web plugin
      sudo pip2 install pyacoustid # for chromaprint/acoustid plugin
        sudo pacman -S chromaprint # for chromaprint/acoustid plugin

      sudo pacman -S gstreamer0.10-python # for (crashy) bpd plugin
        sudo pacman -S gstreamer0.10-bad-plugins
        sudo pacman -S gstreamer0.10-base-plugins
        sudo pacman -S gstreamer0.10-ffmpeg
        sudo pacman -S gstreamer0.10-good-plugins
        sudo pacman -S gstreamer0.10-ugly-plugins
        sudo pacman -S gstreamer0.10-vaapi

    sudo pacman -S mpd

    mkdir -p ~/.config/mpd/playlists
    touch ~/.config/mpd/{database,log,pid,state,sticker.sql}

    ls ~/.config/mpd/mpd.conf ||
      cp /usr/share/doc/mpd/mpdconf.example ~/.config/mpd/mpd.conf &&
      vim ~/.config/mpd/mpd.conf

    systemctl --user enable mpd.service
    systemctl --user start mpd.service

    sudo pacman -S mpc
    sudo pacman -S ncmpcpp

    sudo pacman -S gmpc  # also works well with mopidy
    sudo pacman -S ario
    sudo pacman -S sonata
    pacaur -Sy gbemol

    # mpd client for android - MPDroid https://github.com/abarisain/dmix

This is a native HipChat client (but it's better to just use the web one):

    pacaur -Sy hipchat

Zeal (Dash-like docset viewer):

    pacaur -Sy zeal-git

E-books:

    sudo pacman -S calibre
    sudo pacman -S mcomix

Terminal sharing:

    pacaur -Sy tmate

Kindle:

Installing Kindle under Wine provides a source from which the DeDRM plugin
can extract liberated e-books.

These installers are required:

* [Kindle for PC](http://www.amazon.co.uk/gp/kindle/pc/)
* [ActivePython 2.7.X for Windows (x86)](http://www.activestate.com/activepython/downloads)
* [PyCrypto 2.1 for 32bit Windows and Python 2.7](http://www.voidspace.org.uk/python/modules.shtml#pycrypto)
* [DeDRM tools](http://apprenticealf.wordpress.com/)

And should be installed as follows:

    sudo pacman -S wine wine_gecko wine-mono winetricks
    WINEARCH=win32 winecfg # to set up a default wineprefix (leave it as Windows XP)
    winetricks # select default wineprefix, install component vcrun2008
    wine KindleForPC-installer.exe # register with Amazon account
    msiexec /i ActivePython-2.7.6.9-win32-x86.msi
    wine pycrypto-2.1.0.win32-py2.7.exe

E-book files will appear under `~/My Kindle Content/`.

The DeDRM Calibre plugin is `tools_v*.zip/DeDRM_calibre_plugin/DeDRM_plugin.zip`
(additional installation info in `tools_v*.zip/DeDRM_calibre_plugin/ReadMe_First.txt`).


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
    pacaur -Sy mysql-workbench

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

    pacaur -Sy uuid
    pacaur -Sy postgresql-uuid-ossp
    sudo systemctl restart postgresql

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

    pacaur -Sy multibit
    pacaur -Sy electrum python2-zbar
    pacaur -Sy sx-git
    pacman -S tor vidalia
    pacaur -Sy pybitmessage

Android file access:

    sudo pacman -S gvfs gvfs-mtp

Android dev env:

    # install sdk and platform
    pacaur -Sy android-sdk
    pacaur -Sy android-sdk-platform-tools
    pacaur -Sy android-sdk-build-tools
    pacaur -Sy android-platform

    # make sdk usable as a normal user
    sudo groupadd sdkusers
    sudo gpasswd -a chris sdkusers
    sudo chown -R :sdkusers /opt/android-sdk
    sudo chmod -R g+w /opt/android-sdk/

    # IDE
    pacaur -Sy android-studio

    # Android Debug Bridge
    sudo pacman -S android-tools
    sudo pacman -S android-udev
    sudo gpasswd -a chris adbusers

    # extra Java build stuff
    sudo pacman -S maven

Printer/scanner setup:

    sudo pacman -S cups ghostscript gsfonts avahi
    pacaur -Sy epson-inkjet-printer-stylus-office-tx610fw-series
    sudo systemctl start avahi-daemon.service
    sudo systemctl enable avahi-daemon.service
    sudo systemctl start org.cups.cupsd.service
    sudo systemctl enable org.cups.cupsd.service
    sudo systemctl start cups-browsed.service
    sudo systemctl enable cups-browsed.service
    sudo pacman -S system-config-printer python-pysmbc
    # then run GUI app "Print Settings"

    pacaur -Sy iscan iscan-plugin-network
    sudo vim /etc/sane.d/epkowa.conf  # add line 'net {IP_OF_SCANNER}'
    # then run GUI app "Image Scan! for Linux"

    # EPSON STYLUS PHOTO RX560
    sudo pacman -S gutenprint

Video editing:

    sudo pacman -S avidemux-cli avidemux-gtk

OCR (see [guide on the wiki](https://wiki.archlinux.org/index.php/List_of_applications/Documents#OCR_software)):

    sudo pacman -S tesseract tesseract-data-eng
    pacaur -Sy gscan2pdf
    pacaur -Sy pdftk


PDF editing

    pacaur -Sy masterpdfeditor   # closed source, free for non-commercial use

Google Earth

    pacaur -Sy google-earth
    sudo pacman -S lib32-intel-dri

Emacs

    sudo pacman -S emacs

Crate.io

    pacaur -Sy crate
    sudo systemctl start crate
    sudo systemctl status crate
    open http://localhost:4200/admin

VNC and remote desktop:

    sudo pacman -S libvncserver freerdp remmina
    # install order is important and a reboot may be needed for remmina to show all protocols

Photos:

    sudo pacman -S digikam kipi-plugins kdemultimedia-mplayerthumbs phonon-qt{4,5}-vlc

Gnuplot:

    sudo pacman -S gnuplot
    # copy emacs config from /usr/share/emacs/site-lisp/dotemacs if appropriate

Another PDF reader (as alternative to crashy GNOME Evince):

    sudo pacman -S kdegraphics-okular

OpenVPN client:

    sudo pacman -S networkmanager-openvpn

BitTorrent client:

    pacaur -Sy qbittorrent

qBittorrent can be configured to use a specific interface, such as tun0,
however, if that is unavailable on start, it will fall back to the default
interface.

    sudo vim /usr/share/applications/qBittorrent.desktop # set Exec as follows...

    Exec=bash -c "ls /sys/class/net/tun0 && qbittorrent %U || notify-send 'The tun0 interface is not up'"

An alternative bittorrent client:

    sudo pacman -S deluge pygtk librsvg

Rust programming language:

    pacaur -Sy rust-nightly-bin
    pacaur -Sy libtinfo  # for building rust from source

Scala programming language:

    sudo pacman -S scala
    # a JRE is needed too

Extra fonts: copy `*.otf` files into `~/.fonts` and run `fc-cache`.

Font viewer:

    pacaur -Sy gnome-specimen

Consider adjusting DPI settings in `dconf-editor` under `/org/gnome/desktop/interface/text-scaling-factor` and `scaling-factor`.
Or, use `xrandr`'s `--scale` or `--transform` options.

Install user files and tools, into home directory, including:

* Ruby (via `rbenv`)
* NodeJS (via `nvm`)
* `autoenv`
* Powerline

Add extra startup applications as `*.desktop` files in `.config/autostart/`.

Set up keyboard mapping and caps lock remap:

In `/etc/X11/xorg.conf.d/10-evdev.conf`, replace the keyboard section with:

    Section "InputClass"
            Identifier "evdev keyboard catchall"
            MatchIsKeyboard "on"
            MatchDevicePath "/dev/input/event*"
            Driver "evdev"
            Option "Xkb_Layout" "us"
            Option "Xkb_Variant" "altgr-intl"
            Option "XkbOptions" "ctrl:nocaps"
    EndSection

The `altgr-intl` variant of US keyboard layout allows various European letters
and symbols to be typed. Refer to [this diagram](http://dry.sailingissues.com/keyboard-US-International2.png).

As stated on the wiki page [Keyboard configuration in Xorg](https://wiki.archlinux.org/index.php/Keyboard_configuration_in_Xorg),
GNOME will override some Xkb settings, so:

1. Set `ctrl:nocaps` with `dconf-editor` as described on the wiki under [GNOME > Modify Keyboard with XkbOptions](https://wiki.archlinux.org/index.php/GNOME#Modify_Keyboard_with_XkbOptions)
2. Add a keyboard layout ("English (international AltGr dead keys)") via `Settings > Keyboard > Input Sources > Input Sources > +`

For multiple keyboards, see: [Two keyboards on one computer](http://superuser.com/questions/75817/two-keyboards-on-one-computer-when-i-write-with-a-i-want-a-us-keyboard-layout/294034#294034).


Backup:

    sudo pacman -S duplicity python2-boto

Accounting:

    pacaur -Sy ledger
    pacaur -Sy gnucash-xbt

## Issues

* 2015-04-11, after upgrading GNOME from 3.14 to 3.16
  [Mouse cursor missing](https://wiki.archlinux.org/index.php/GNOME#Mouse_cursor_missing)
  It happens when the screen has been off. This thread seems related:
    [](https://bugs.launchpad.net/ubuntu/+source/gnome-settings-daemon/+bug/1238410)
  Pressing Ctrl-Alt-Backspace seems to bring the cursor back (or switching user
  back to the current user).

* Noticed on 2015-05-09 that the trackpad doesn't do two-finger horizontal
  scroll. Also it's really sensitive and turning it off in GNOME doesn't work.

* Since update around 2015-04-12, gnome going idle makes the mouse pointer
  disappear. https://bbs.archlinux.org/viewtopic.php?id=110023

* Tearing reappeared at some point. Try alternative fix.

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
  * https://wiki.archlinux.org/index.php/HiDPI

## TODO

* try mopidy
* subpixel rendering and autohinting with https://wiki.archlinux.org/index.php/Infinality
  read: http://www.binarytides.com/gorgeous-looking-fonts-ubuntu-linux/
* for making boot USBs: http://unetbootin.sourceforge.net/
* check out dm-crypt
* sort old data
* consider switching to a https mirror list via https://www.archlinux.org/mirrorlist/
* finish chrony for time sync
* cron alternative for intermittent processing/network
* add back skype with some lockdown
* Write PKGBUILDs for:
  - [prax](https://github.com/ysbaddaden/prax) (needs systemd service)
  - [pdfocr](https://github.com/gkovacs/pdfocr/)
* Clipboard manager: [http://parcellite.sourceforge.net/?page_id=2]()
* encrypting the boot partition: http://www.pavelkogan.com/2014/05/23/luks-full-disk-encryption/
* Check out Filebot (TV and movie renamer)
* Check out [XFCE recommended apps](http://wiki.xfce.org/recommendedapps)

* read [Arch wiki page for the ASUS Zenbook Prime UX31A](https://wiki.archlinux.org/index.php/ASUS_Zenbook_Prime_UX31A)
* https://wiki.archlinux.org/index.php/General_Recommendations
* https://wiki.archlinux.org/index.php/Laptop
* https://wiki.archlinux.org/index.php/lm_sensors, https://wiki.archlinux.org/index.php/Fan_Speed_Control

