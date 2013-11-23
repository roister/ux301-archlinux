# Kiste

Readable, repeatable dev box management.


## Get started with Kiste

After OS install (see below), software and config can be set up with kiste
as follows:

    # get the files
    cd ~
    wget https://github.com/chrisberkhout/kiste/archive/master.zip -O kiste-master.zip
    unzip kiste-master.zip
    rm kiste-master.zip
    mv kiste-master kiste
    cd kiste

    # get kiste running
    ./kiste-bootstrap

    # manually place your SSH key, have kiste check and correct the permissions
    mkdir -p ~/.ssh
    cat > ~/.ssh/id_rsa
    cat > ~/.ssh/id_rsa.pub
    ./kiste usr/ssh-key.yml

    # git-ize kiste
    git init .
    git remote add origin git@github.com:chrisberkhout/kiste.git
    git fetch
    git checkout master --force  # discards any chanages in working dir

    # build the the box
    ./kiste


## OS install and resolved issues

These notes relate to running Xubuntu 13.10 Saucy on `MacBookPro8,2`.

### Create distribution install media and boot it (manual)

* http://www.ubuntu.com/download/desktop/create-a-usb-stick-on-mac-osx

### Installation graphics support using the integrated graphics card (manual)

### Discrete graphics activation via EFI stub boot method (manual)

### Pin/remove grub so update doesn't overwrite refind (manual)

### Activate HDMI audio output (manual)

### Switch wireless driver (manual)

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

### Fix screen tearing with compton compositor (via kiste)

### Apply remembered display config via a script (via kiste)

XFCE desktop doesn't remember different display configurations, but this can be
[scripted](https://github.com/chrisberkhout/dot-other/blob/master/bin/fix-attachments.sh)
using `xrandr` and `arandr`.


## Remaining issues

Some of these are hardware/Linux related, others are probably just shell/editor setup.

### Tearing reappears after display changes

There is a useful test pattern [on YouTube](www.youtube.com/watch?v=ceX18O9pvLs).

This isn't fixed by any of the following:

    killall compton && sleep 1 && compton -b
    xfdesktop --reload
    killall -HUP xfdesktop

It is fixed by logging out and back in.
It is fixed by going to "Switch User" (even switching immediately back to the same user).

It would be good to be able to trigger this reset from the command line.

### Internet doesn't work well

This issue has persisted after changing wireless drivers as described above. It
also seems to happen with wired connections.

It shows hostname resolution issues. It may be a networking issue rather than a
hardware one.

### Apple bluetooth keyboard

Sometimes won't connect. Here are some relevant resources:

* http://askubuntu.com/questions/12502/how-do-i-get-the-apple-wireless-keyboard-working-in-10-10
* http://askubuntu.com/questions/148981/how-do-i-use-a-bluetooth-mac-keyboard-on-ubuntu

Sometimes sticks on keys. Last successful resolution:

* Shut down laptop
* Take out and put back in keyboard batteries
* Boot laptop

Sometimes disconnects.

### No easy switching between discrete and integrated graphics

This would be useful for extending battery life.

### Apple IR remote control

Parole Media Player recognises play/pause.
VLC recognises nothing.

### Terminal font resize from keyboard

This doesn't seem to be possible with `xfce4-terminal`.

### Bash in tmux sometimes splits commands

For example:

* `ls` expands to `ls --color=auto`, but then runs as separate commands
  `ls` and `--color=auto`.
* `git status` runs as `git` and `status`.

It only happens in tmux.
Double sourcing the startup files in plain bash doesn't produce the issue.

[Discussion on the tmux mailing list](http://sourceforge.net/mailarchive/forum.php?thread_name=3rl7x75rnx2d7gu396ju2lyd.1384763818147%40email.android.com&forum_name=tmux-users)

### Problems after stopping a long listing with `<C-c>`

Sometimes, one of the folllowing occurs:

* Terminal stops echoing typing.
  The terminal must be closed.

* The tmux prefix stops working.
  Can be worked around by reattaching in a new teriminal.

### ctrlp.vim freezes

When trying to open a file in a split with `<C-s>` (instead of `<C-x>`).
Killing vim doesn't seem to work, I need to kill the pane using tmux.


## Questions

* Adding PPA source for different distro release.
* Ignore some stuff from a PPA (e.g. when one PPA has multiple things and you only want one).
* APT keys - do they matter?
* Where to put downloaded archives extracted into `/opt` and `/usr/local/src`.


## TODO

* Heroku toolbelt: https://toolbelt.herokuapp.com/debian
* Alternative to pow rack dev server: https://github.com/ysbaddaden/prax
* http://askubuntu.com/questions/68809/how-to-format-a-usb-or-external-drive
* Clipboard manager: http://parcellite.sourceforge.net/?page_id=2
* https://apps.ubuntu.com/cat/applications/d4x/
* Check out http://wiki.xfce.org/recommendedapps

