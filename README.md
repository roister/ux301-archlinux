# Kiste

Readable, repeatable dev box management.

## Get started

    # get the files
    cd ~
    wget https://github.com/chrisberkhout/kiste/archive/master.zip -O kiste.zip
    unzip kiste.zip
    rm kiste.zip
    cd kiste

    # basic setup
    ./bin/kiste-bootstrap

    # manually paste in your SSH key and have kiste check and correct the permissions
    mkdir -p ~/.ssh
    cat > ~/.ssh/id_rsa
    cat > ~/.ssh/id_rsa.pub
    ./bin/kiste usr/ssh-key

    # build the the box
    ./bin/kiste

## TODO

* Convert kiste dir into git repo if created from zip
* Alert about books missing from all.yml

