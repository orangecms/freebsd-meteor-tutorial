## About
This is a tutorial on running Meteor applications on FreeBSD servers, e.g. [droplets on DigitalOcean](https://www.digitalocean.com/company/blog/presenting-freebsd-how-we-made-it-happen/). If they accept it, DigitalOcean might publish this article in their tutorials area as well. But first of all I want it to be available to people, regardless of the hosting platform, so kudos to GitHub for simply allowing this. :)

## Introduction

Meteor works with legacy Node.js (v0.10.x). The pkg repository, however, offers the latest version.

## Prerequisites on the server

### Research
Let's consult Google. We end up in [the FreeBSD forums](https://forums.freebsd.org/threads/52004/).

So there is [a port for npm](http://www.freshports.org/www/npm) (a description how to build a package).

### Preparation
First things first. We need to upgrade the ports tree, so let's [RTFM](https://www.freebsd.org/doc/handbook/ports-using.html) how to do that again.

Assuming we already had a tree:
```bash
portsnap fetch update
```

### Configuration
First cd to the npm port:
```bash
cd /usr/ports/www/npm
```

And then use `NODE010` as mentioned in the forum. [The man page says](https://www.freebsd.org/cgi/man.cgi?ports(7)) `config` lets us change the OPTIONS. When running
```bash
make config
```
, we get a simple ncurses menu. Choose `NODE010` by navigating with the arrow keys and pressing space to select, then Return.

*Note*: This requires interaction. On a CI server, I wouldn't want that, but couldn't figure out how to do this non-interactively. PRs are welcome!

To make sure we got it right:
```bash
make showconfig
```
The output should contain the following line:
```bash
NODE010=on: Use www/node010 as backend
```

### Installation
Now as usual:
```bash
make && make install
```

Oh wait, that doesn't work if we already have npm installed:
```bash
===>  Checking if npm already installed
===>   npm-3.8.6 is already installed
      You may wish to ``make deinstall'' and install this port again
      by ``make reinstall'' to upgrade it properly.
```

If installed through pkg, simply remove it through it, together with Node.js:
```bash
pkg remove node npm
```

And try again:
```bash
make install
```

Now this builds Node v0.10.44 (as of writing this) and installs it as well. :)

### Summary
The whole procedure boils down to these few commands:

```bash
portsnap fetch update
cd /usr/ports/www/npm
make config
pkg remove node npm
make && make install
```

## Build & Deploy

### Create a tarball
In the project directory where you usually run `meteor`, run:
```bash
meteor build .
```

This will create a `.tar.gz` file, usually named `my_app.tar.gz`.

### Upload the app to the server
```bash
ssh-add ~/.ssh/my-ssh-key-to-my-server
sftp freebsd@my-domain.org
put my_app.tar.gz
```

### Deploy the app
```bash
ssh freebsd@my-domain.org
tar -xf /home/freebsd/my_app.tar.gz            # extract
cd bundle/programs/server; npm i; cd ../../../ # install node modules
```

From here on, the app could already be run by simply invoking `node my_app/main.js`. This way you can now confirm everything was installed correctly - at least on the server side.

### Running as a system service

#### Preparation
As a first step, move the app to the global web server directory:
```bash
sudo chown -R root:wheel bundle                # fix permissions
sudo mv bundle /usr/local/www/my_app           # move to web server home
```

Then create directories where log and pid files will be stored. Log files will help you understand what happened when errors occur or something just didn't work, and the pid files are necessary for the service.
```
sudo mkdir -p /var/log/node /var/run/node      # create dirs for log and pid files
```

#### Creating a service
##### Init script
[FreeBSD rc scripts](https://www.freebsd.org/doc/en/articles/rc-scripting/) reside in `/usr/local/etc/rc.d/`. This is where we will create our scripts as well.
```bash
cd /usr/local/etc/rc.d/
```

##### Configuration
Write the environment variables (env vars) into `/etc/rc.conf.d/my_app`:

```bash
my_app_enable=YES                                        # enable for auto-run on server startup and service start instead of onestart
NODE_ENV=production                                      # this is our production environment, check out the Node.js docs for more
PORT=3123                                                # make sure to use one port per app, and remember it for the reverse proxy
ROOT_URL=https://my-domain.org/my_app                    # the root URL as seen from outside, used by Meteor.absoluteUrl
MONGO_URL=mongodb://meteor:meteor@127.0.0.1:27017/my_app # find another tutorial if you need to set up MongoDB ;)
```
