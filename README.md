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

### Display scrambling issue

Sometimes when changing displays, waking from sleep or doing switch user, the
graphics gets all scrambled. Sometimes this can be recovered from by sleeping
and waking again, other time it couldn't be recovered.

### Fans constantly running

[https://help.ubuntu.com/community/MacBookPro8-2/Quantal#Sensors]()

### Internet doesn't work well

This issue has persisted after changing wireless drivers as described above. It
also seems to happen with wired connections.

It often seems to fail on hostname lookup, so may be a DNS issue.

Some releated resources:

* [http://askubuntu.com/questions/152593/command-line-to-list-dns-servers]()
* [https://developers.google.com/speed/public-dns/]()
* [http://www.opendns.com/technology/opendns-vs-google-public-dns/]()
* [Ubuntu forums: Extremely slow internet with Ubuntu, esp. DNS lookup times](http://ubuntuforums.org/showthread.php?t=1487409)
* [ask ubuntu: 10.10 - Slow DNS Resolution](http://askubuntu.com/questions/8704/slow-dns-resolution)
* [ask ubuntu: 12.10 - Extrememly slow DNS lookup](http://askubuntu.com/questions/272358/extrememly-slow-dns-lookup)
* [Ubuntu forums: DNS related slow connection](http://ubuntuforums.org/showthread.php?t=1778622)

Maybe it is just after sleep, and could be resolved with this technique
(described for a different driver):

* [SOLVED - No wireless connections after sleep/suspend](http://ubuntuforums.org/showthread.php?t=2004690)

### Apple bluetooth keyboard

Sometimes won't connect. Here are some relevant resources:

* [http://askubuntu.com/questions/12502/how-do-i-get-the-apple-wireless-keyboard-working-in-10-10]()
* [http://askubuntu.com/questions/148981/how-do-i-use-a-bluetooth-mac-keyboard-on-ubuntu]()

Sometimes sticks on keys. Restarting the bluetooth service doesn't help. The
last successful resolution was:

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
  This one also happened when doing mouse select in tmux, then `<tmux,y>`.

### In tmux, left and right clicks paste unwanted characters

For example: `N O#` or `"C#` or `B B#` or `B"B#`.
This happens when the mouse it outside of the area where it works (ie. on the right side of a huge screen).

### ctrlp.vim freezes

When trying to open a file in a split with `<C-s>` (instead of `<C-x>`).
Killing vim doesn't seem to work, I need to kill the pane using tmux.


## Questions

* Adding PPA source for different distro release.
* Ignore some stuff from a PPA (e.g. when one PPA has multiple things and you only want one).
* APT keys - do they matter?
* Where to put downloaded archives extracted into `/opt` and `/usr/local/src`.


## TODO

* draft notes
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

## Extra notes from elsewhere

* [http://www.techrepublic.com/blog/apple-in-the-enterprise/how-to-create-a-bootable-usb-to-install-os-x/]()
* Encryption:
  * [http://askubuntu.com/questions/342274/how-do-i-create-an-encrypted-growable-disk-image?rq=1]()
  * [http://emount.sourceforge.net/?page=screenshots]()
* Commercial software for Mac access to ext3/4:
  * [http://www.paragon-software.com/home/extfs-mac/]()
* bypassing hfs+ permissions:
  * [http://bindfs.org/]()
  * [http://superuser.com/questions/387284/by-passing-default-permissions-when-mounting-hfs-volumes-in-linux]()
  * [http://askubuntu.com/questions/100167/how-to-mount-hfs-drive-and-ignore-permissions]()
* Mac tweaks script `UbuntuOneiricMacBookPro8,2.sh`:
  * [https://help.ubuntu.com/community/MacBookPro8-2/Oneiric?action=AttachFile&do=view&target=UbuntuOneiricMacBookPro8%2C2.sh]()


