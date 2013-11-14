# Kiste

Readable, repeatable dev box management.


## Get started with Kiste

    # get the files
    cd ~
    wget https://github.com/chrisberkhout/kiste/archive/master.zip -O kiste-master.zip
    unzip kiste-master.zip
    rm kiste-master.zip
    mv kiste-master kiste
    cd kiste

    # get kiste running
    ./bin/kiste-bootstrap

    # manually place your SSH key, have kiste check and correct the permissions
    mkdir -p ~/.ssh
    cat > ~/.ssh/id_rsa
    cat > ~/.ssh/id_rsa.pub
    ./bin/kiste usr/ssh-key

    # git-ize kiste
    git init .
    git remote add origin git@github.com:chrisberkhout/kiste.git
    git fetch
    git checkout master --force  # discards any chanages in working dir

    # build the the box
    ./bin/kiste


## OS install

These notes relate to `MacBookPro8,2`.

### Create distribution install media and boot it

* http://www.ubuntu.com/download/desktop/create-a-usb-stick-on-mac-osx

### Use integrated graphics for install

### Use discrete graphics via EFI stub boot method

### Activate HDMI audio output

### Wireless

The `MacBookPro8,2` has BCM4331. For improved wireless reliability, remove the
`wl` "Proprietary Broadcom STA Wireless driver" (package: `bcmwl-kernel-source`)
and use the b43 driver. Use to the documentation:

* [Ubuntu: WifiDocs/Driver/bcm43xx](https://help.ubuntu.com/community/WifiDocs/Driver/bcm43xx#b43%20-%20Internet%20access) (for instructions)
* [Linux Wireless wiki](http://wireless.kernel.org/en/users/Drivers/b43) (for background)

The procedure is:

    sudo apt-get update
    sudo apt-get install firmware-b43-installer
    sudo modprobe -r b43 ssb wl brcmfmac brcmsmac bcma
    sudo modprobe b43
    # wait... try disabling and reenabling wi-fi through the GUI
    sudo apt-get purge bcmwl-kernel-source  # so this doesn't take over again

### Pin/remove grub so it doesn't reinstall itself on every update and overwrite refind

### Add compton compositor to fix screen tearing issue


## TODO

* Alternative to pow rack dev server: https://github.com/ysbaddaden/prax
* Clipboard manager: http://parcellite.sourceforge.net/?page_id=2
* Check out http://wiki.xfce.org/recommendedapps
* Alert about books missing from all.yml

