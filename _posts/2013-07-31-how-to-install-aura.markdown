---
layout: post
title:  "How To Install aura"
date:   2013-07-31 23:57:00
---

## Prerequisites

If you want to use [aura](https://www.github.com/fosskers/aura), you first have to be using Pacman, the package manager for Archlinux. The easiest way to do that? Just use Arch!  It's my favorite distribution and I'm getting all my friends into it. It's the cool thing to do!

### Now ya got Arch

Since you now have Arch, installing aura is kind of funny.  Since it lives on the AUR (Arch User Repository), you can't install it directly through Pacman.  But thanks to some awesome folks, all of the Haskell packages are up-to-date on an unofficial Pacman repository.

1.  First add the following code between the [core] and [extra] section in /etc/pacman.conf

        [haskell-core]
        Server = http://www.kiwilight.com/haskell/core/$arch

2.  Resync the packages.  This should fail with the error that it can't sign the haskell-core repository.

        sudo pacman -Sy

3.  Sign the new database.

        sudo pacman-key -r 4209170B
        sudo pacman-key --lsign-key 4209170B

4.  Resync again.  This should now work!

        sudo pacman -Sy

5.  Pull down the aura package.

        curl https://aur.archlinux.org/packages/au/aura/aura.tar.gz -o aura.tar.gz

6.  Ungzip and tar the package.

        tar -xzf aura.tar.gz

7.  Change into the aura directory and make the package with all it's dependencies.

        cd aura
        makepkg -s

8.  When that is done, simply install the locally built package.

        sudo pacman -U aura-1.2.0.2-1-i686.pkg.tar.xz

You should be all set now!  Have fun with aura!
