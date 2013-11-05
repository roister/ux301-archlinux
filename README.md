# Kiste

Readable, repeatable dev box management.

## Get started

    # get the files
    cd ~
    wget https://github.com/chrisberkhout/kiste/archive/master.zip -O kiste.zip
    unzip kiste.zip
    rm kiste.zip
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

## TODO

* Alert about books missing from all.yml

