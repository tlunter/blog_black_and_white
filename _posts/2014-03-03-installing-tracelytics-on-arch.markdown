---
layout: post
title:  "Installing Tracelytics on Arch"
date:   2014-03-03 00:07:00
---

If anyone's seeing the beautiful charts that comes from AppNeta's Tracelytics, you know how beneficial it can be.  After refreshing various pages around my website and seeing some delays, I really wanted to get to the bottom of it.

After a quick weekday night tweet, one of AppNeta's employees reached out and noted that their cheapest solution is free. I really wanted to get this installed, and figured I'd try it out past this weekend.

The signup was smooth, but the install file I downloaded only supported Ubuntu, Debian or Red Hat/Fedora.  Since I run Arch, I needed to modify the script.

# Step one: Get the files

Unfortunately, the Debian and RPM packages for Oboe and the Tracelyzer aren't immediately accessible.  That means this isn't going to just be modifying the installer, and really needs manual intervention.

Instead of just hosting the debian packages somewhere cURL can get them, these packages are just hosted behind apt and yum repository walls.  To get the packages, I needed to install aptitude on Arch.  There is [apt](https://aur.archlinux.org/packages/apt), but of course the PKGBUILD was out of date.  So I updated that (added a comment on the AUR package and it's already fixed).

After upgrading the package version and including the libapt-private.so library to /usr/lib, I was able to get aptitude working.  I used `apt-get download liboboe0 liboboe-dev tracelyzer` to get the files I needed and copied these to my server.

# Step two: Extract the files

Debian packages are pretty simple to extract if you have `dpkg` installed.  [dpkg](https://aur.archlinux.org/packages/dpkg) has very little dependencies so getting it installed from the AUR is simple (and it's up to date).  `dpkg -x FILE` extracts the file structure returning where all the files would exist relative to root, so a simple copy puts those in place.  

In addition, debian packages has post install scripts that can be run.  Using `dpkg-deb --control FILE TARGET`, you can extract these.  Installing liboboe0 and liboboe-dev were as simple as moving the files and running `ldconfig`.  The tracelyzer agent has a bit more going on.

Following the standard for most linux applications, the install adds a user:group called `appneta:appneta` and sets the owner of application specific files.  It used addgroup and adduser which aren't available on Arch, so I switched those to useradd and groupadd.  So I modified the script and ran it standalone.

From

    #!/bin/sh
    # postinst script for tracelyzer
    #
    # see: dh_installdeb(1)
    
    set -e
    
    # summary of how this script can be called:
    #        * <postinst> `configure' <most-recently-configured-version>
    #        * <old-postinst> `abort-upgrade' <new version>
    #        * <conflictor's-postinst> `abort-remove' `in-favour' <package>
    #          <new-version>
    #        * <postinst> `abort-remove'
    #        * <deconfigured's-postinst> `abort-deconfigure' `in-favour'
    #          <failed-install-package> <version> `removing'
    #          <conflicting-package> <version>
    # for details, see http://www.debian.org/doc/debian-policy/ or
    # the debian-policy package
    
    # First: rename tracelyt user/group to appneta, if they exist
    # if appneta user/group already exists, leave both
    if getent group tracelyt >/dev/null; then
      if ! getent group appneta >/dev/null; then
        groupmod -n appneta tracelyt
      fi
    fi
    if getent passwd tracelyt >/dev/null; then
      if ! getent passwd appneta >/dev/null; then
        usermod -l appneta -c "AppNeta Daemon" tracelyt
      fi
    fi
    
    if ! getent group appneta >/dev/null; then
        # Adding system group: appneta.
        addgroup --system appneta
    fi
    
    if ! getent passwd appneta >/dev/null; then
        # Adding system user: appneta.
        adduser \
          --system \
              --disabled-login \
          --ingroup appneta \
          --home /var/log/tracelyzer \
              --no-create-home \
          --gecos "AppNeta Daemon" \
          --shell /bin/false \
          appneta
    fi
    
    chown appneta:appneta /var/log/tracelyzer
    chown appneta:appneta /var/run/tracelyzer
    chown appneta:appneta /var/lib/tracelyzer
    chown appneta:appneta /etc/tracelyzer
    chmod 0700 /var/log/tracelyzer
    chmod 0700 /var/run/tracelyzer
    chmod 0755 /var/lib/tracelyzer
    chmod 0755 /etc/tracelyzer
    if [ ! -f /etc/tracelytics.conf ]; then
        touch /etc/tracelytics.conf
    fi
    chown appneta:appneta /etc/tracelytics.conf
    /usr/sbin/appneta-config -s 
    
    case "$1" in
        configure)
        ;;
    
        abort-upgrade|abort-remove|abort-deconfigure)
        ;;
    
        *)
            echo "postinst called with unknown argument \`$1'" >&2
            exit 1
        ;;
    esac
    
    # dh_installdeb will replace this with shell code automatically
    # generated by other debhelper scripts.
    
    # Automatically added by dh_installinit
    if [ -x "/etc/init.d/tracelyzer" ]; then
            if [ ! -e "/etc/init/tracelyzer.conf" ]; then
                    update-rc.d tracelyzer defaults >/dev/null
            fi
            invoke-rc.d tracelyzer start || exit $?
    fi
    # End automatically added section
    # Automatically added by dh_makeshlibs
    if [ "$1" = "configure" ]; then
            ldconfig
    fi
    # End automatically added section

    exit 0

to

    #!/bin/sh
    # postinst script for tracelyzer
    
    set -e
    
    # First: rename tracelyt user/group to appneta, if they exist
    # if appneta user/group already exists, leave both
    if getent group tracelyt >/dev/null; then
      if ! getent group appneta >/dev/null; then
        groupmod -n appneta tracelyt
      fi
    fi
    if getent passwd tracelyt >/dev/null; then
      if ! getent passwd appneta >/dev/null; then
        usermod -l appneta -c "AppNeta Daemon" tracelyt
      fi
    fi
    
    if ! getent group appneta >/dev/null; then
        # Adding system group: appneta.
        groupadd --system appneta
    fi
    
    if ! getent passwd appneta >/dev/null; then
        # Adding system user: appneta.
        useradd \
          --system \
          -g appneta \
          --home /var/log/tracelyzer \
          -c "AppNeta Daemon" \
          --shell /bin/false \
          appneta
    fi
    
    chown appneta:appneta /var/log/tracelyzer
    chown appneta:appneta /var/run/tracelyzer
    chown appneta:appneta /var/lib/tracelyzer
    chown appneta:appneta /etc/tracelyzer
    chmod 0700 /var/log/tracelyzer
    chmod 0700 /var/run/tracelyzer
    chmod 0755 /var/lib/tracelyzer
    chmod 0755 /etc/tracelyzer
    if [ ! -f /etc/tracelytics.conf ]; then
        touch /etc/tracelytics.conf
    fi
    chown appneta:appneta /etc/tracelytics.conf
    /usr/sbin/appneta-config -s
    
    exit 0

So now, this should run and since all the files have been moved beforehand, it should succeed.

## libcurl hairiness

At least, this should work in theory.  And for the most part, it does.  Until the last command.

    appneta-config: /usr/lib/libcurl.so.4: version `CURL_OPENSSL_3' not found (required by appneta-config)

`appneta-config` is built against a different version of libcurl. Because it's a different version, different versioned symbols exist and in this version, only CURL\_OPENSSL\_4 exists.

After a bit of googling, I find downgrading the cURL package removes the CURL\_OPENSSL\_3 error.  Instead of playing it safe and actually installing the older version of cURL, I swap out the libcurl file and run `appneta-config -s` as root, and it works.  After, I swap it back with the known good version.

# Step three: Start up

The tracelyzer package comes with an /etc/init.d script for upstart.  Since I'm using Arch, I use systemd.  Since there is a lot of configuration going on in this file, I decide I need to do something really strange.  I move all the configuration to `/etc/conf.d/tracelyzer` and then I create two bash files in /usr/bin.  Both of them contain almost nothing, but source the config file and run `$DAEMON $DAEMON_OPTS` and `$DAEMON2 $DAEMON2_OPTS`.  Since these variables were set up in the config file, they run perfectly.

`$DAEMON` is the `tracelyzer` program and `$DAEMON2` is `tracelyzer-conn`.  I make two systemd services with `tracelyzer` being a forking process and `tracelyzer-conn` being a simple one.

AppNeta reports seeing my tracelyzer agents and all looks great.  Smooth sailing!

# Step four: Recompiling nginx

Since the tracelyzer agent is communicating with AppNeta, I need to now communicate with the tracelyzer agent.  I use nginx as my external http server, and the plugin support for nginx has to be compiled in.

The AUR again has an [nginx-custom](https://aur.archlinux.org/packages/nginx-custom) package that has the standard nginx package and few extra third party addons.  Modifying the PKGBUILD, I can add in the nginx_oboe plugin.

I just had to add:

    _oboe_ver="latest"
    _oboe_dirname="ngx_oboe"

with the rest of the 3rd party modules,

    ${_oboe_dirname}.tar.gz::https://files.tracelytics.com/src/nginx_oboe-${_oboe_ver}.tar.gz"

with the sources,

    'ba93cff98fa5ad34dfeccf0bab6a6c15'

to the end of the md5sums.

In the build() section, I had to move the files:

    mv nginx_oboe* ${_oboe_dirname}

and add the module to the ./configure command

    --add-module=../${_oboe_dirname}

All looks to be running clean.

Until, once again, crippling news.  Instead of compiling correctly, the linker fails due to 

    liboboe.so: undefined reference to `RAND_pseudo_bytes@OPENSSL_1.0.0'

Running `ldd` on `/usr/lib/liboboe.so` returns:

    /usr/lib/liboboe.so: /usr/lib/libcrypto.so.1.0.0: no version information available (required by /usr/lib/liboboe.so)
        linux-vdso.so.1 (0x00007fffc5353000)
        libcrypto.so.1.0.0 => /usr/lib/libcrypto.so.1.0.0 (0x00007f7550d32000)
        libpthread.so.0 => /usr/lib/libpthread.so.0 (0x00007f7550b15000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f755076d000)
        libdl.so.2 => /usr/lib/libdl.so.2 (0x00007f7550568000)
        libz.so.1 => /usr/lib/libz.so.1 (0x00007f7550352000)
        /usr/lib64/ld-linux-x86-64.so.2 (0x00007f755134f000)

Now, I'm not sure where to go from here.  It looks like liboboe is calling a versioned symbol function from libcrypto but Arch's version of libcrypto doesn't have these versioned symbols and fails. After a brief twitter conversation with AppNeta's developer evangelist [@jmeickle](https://twitter.com/jmeickle), I've found out they might start statically linking libcrypto to prevent this issue.

After rummaging around with objdump and nm, I can see where the method call is happening, just can't fix it (the burden of closed source!).

Since I can't do anything about this, time to move onto Ruby.

# Step five: Oboe in Ruby with Sinatra

Again, since running Arch (I'll tell ya all day, everyday), I am running Ruby 2.1.0 like a good-rolling-release-boy.  Was able to install the gem, but as soon as I try and do a require, I get:

    [oboe/error] Problem loading: #<NoMethodError: undefined method `[]' for nil:NilClass>

Not really sure what is going on here.  I know I can look through the source, but at this point I hit 3 AM and needed some rest.  I plan to follow up with someone at AppNeta/Tracelytics to see if any of this makes sense.

# Postmortem

So it doesn't seem so trivial getting this installed anymore, but I hope this helps pave a way for anyone at AppNeta trying to get this installed and working.  Not building from a Debian package would obvious be the way to go with having access to the build server, and compiling against the latest versions of libcrypto and libcurl would be quick.  Not sure what the issue with Ruby is about, but maybe I can get some answers with Google or their knowledge base.  If I strike out there, I'll hit up twitter and reach out any way I can.
