# Hacking on the MacPorts buildbot

## Setting up both buildbot master and slave for testing on localhost

These steps explain how to install buildbot locally for hacking on the
infrastructure. This will run both the buildbot master and buildbot
slave on localhost. Note that the buildbot slave will run in
a non-default prefix, to avoid interfering with your installation in
`/opt/local`.

For a production setup you would probably want the reverse:

  * let the buildbot slave build ports in `/opt/local`
  * place the auxiliary installation for tools somewhere else, for
    example under `/opt/mports`

The exact locations can be configured with `config.json.sample`.

### Setting up buildbot master on localhost

#### 1. Install buildbot

    sudo port install buildbot


#### 2. Create new directory with buildbot master configuration

    sudo mkdir -p /opt/mp-buildbot
    sudo chown -R $USER:buildbot /opt/mp-buildbot
    sudo chmod -R 775 /opt/mp-buildbot

    buildbot create-master /opt/mp-buildbot/master


#### 3. Add and edit sample configuration files

    cd /opt/mp-buildbot/master
    ln -s .../path/to/contrib/buildbot-test/master.cfg
    cp .../path/to/contrib/buildbot-test/config.json.sample config.json
    cp .../path/to/contrib/buildbot-test/slaves.json.sample slaves.json

Check settings in `config.json` and adapt as needed.


#### 4. Set up authentication

    cd /opt/mp-buildbot/master
    htpasswd -c -d ./htpasswd admin


#### 5. Starting buildbot

To start buildbot, execute the `start` command. The OS X firewall will
request you to allow access for Python. Then you can view the buildbot
instance in your web browser.

    buildbot start /opt/mp-buildbot/master
    open http://localhost:8010/


#### 5. Testing changes

After making any changes to `master.cfg`, you can reload the
configuration with the `reconfig` command. This is faster than doing
a full `restart`. In a similar way, you can completely `stop` the
buildbot.

    buildbot reconfig /opt/mp-buildbot/master
 
    buildbot restart /opt/mp-buildbot/master

    buildbot stop /opt/mp-buildbot/master


### Setting up buildbot slave on localhost

This will use your copy of MacPorts in `/opt/local` for all tooling, but
actual builds on the slave will be made in a separate prefix. Make sure
this installation provides an up-to-date ports tree.

You will need the subversion port in this prefix for tooling, as
`/usr/bin/svn` will have problems validating the Subversion server
certificate due to a well-known bug in Mac OS X >= 10.7.

    sudo port install subversion


#### 1. Install MacPorts into a new prefix

You need to use a new prefix for this installation. These instructions
will use `/opt/mp-buildbot/prefix`.

You will have to install this from source following

* https://guide.macports.org/#installing.macports.source.multiple

    export PATH=/bin:/sbin:/usr/bin:/usr/sbin
    MP_PREFIX=/opt/mp-buildbot/prefix
    ./configure --prefix=$MP_PREFIX --with-applications-dir=$MP_PREFIX/Applications
    make && sudo make install

TODO: what to do about `startupitem_install no` in `$MP_PREFIX/etc/macports/macports.conf`?

#### 2. Install buildbot-slave

Install buildbot-slave in your *normal* `/opt/local` prefix:

    sudo port install buildbot-slave


#### 3. Create new directory with buildbot slave configuration

Create a directory that will contain the buildslaves working directory. 

    BB_PREFIX=/opt/mp-buildbot
    sudo mkdir -p "$BB_PREFIX"
    sudo chown -R $USER:buildbot "$BB_PREFIX"
    sudo chmod -R 775 "$BB_PREFIX"

Create two buildslaves, one for base running as the buildbot user, one
for ports running as root:

    OSXVERS=$(sw_vers -productVersion | grep -oE '^[0-9]+\.[0-9]+')
    ARCH=$(uname -m)
    PASSWORD=... # add your password here

    sudo -H -u buildbot buildslave \
                create-slave --umask 022 \
                "${BB_PREFIX}/slave-base" \
                localhost:9989 \
                base-${OSXVERS}_${ARCH} \
                $PASSWORD
    
    sudo -H buildslave \
                create-slave --umask 022 \
                "${BB_PREFIX}/slave-ports" \
                localhost:9989 \
                ports-${OSXVERS}_${ARCH} \
                $PASSWORD

#### 3. Add new buildbot slaves to buildbot master configuration

Add the buildslaves to your buildmaster's `slaves.json` using the passwords you
provided when creating them. You will also need to change the list of
`'build_platforms'` to the local platform you have set up a build slave for.

    $EDITOR /opt/mp-buildbot/master/slaves.json

IMPORTANT! Change the following configuration options in config.json:

    $EDITOR /opt/mp-buildbot/master/config.json

```json
    "slaveprefix":  "/opt/local",
    "toolsprefix":  "/opt/mp-buildbot/prefix",
```

Reconfigure the buildbot master:

    buildbot reconfig /opt/mp-buildbot/master


#### 4. Start the two buildslaves

These commands start the new build slaves. They should connect to the
master successfully and be visible in the web interface.

    sudo -H -u buildbot buildslave start "${BB_PREFIX}/slave-base"
    sudo -H buildslave start "${BB_PREFIX}/slave-ports"

__TODO: this should be done with plists!!!__

#### 5. Test your first build

TODO


## Setting up buildbot master and slaves for production

In a production environment the slaves run on different machines to produce
MacPorts packages for various platforms. Most of the configuration above
applies to the production environment. The configuration of the master in
config.json should use prefixes such that packages are produced for /opt/local:

> "slaveprefix":  "/opt/local"
> "toolsprefix":  "/opt/mports"

When setting up the slaves, specify the hostname and port of the machine
running the buildbot master instead of "localhost:9989".

## Troubleshooting

# Subversion checkout fails

> svn: E170013: Unable to connect to a repository at URL 'https://svn.macports.org/repository/macports/contrib/mp-buildbot'
> svn: E230001: Server SSL certificate verification failed: issuer is not trusted

Install the `subversion` port to 'toolsprefix'.
