## About
This is a tutorial on running Meteor applications on FreeBSD servers, e.g.
[droplets on DigitalOcean](https://www.digitalocean.com/company/blog/presenting-freebsd-how-we-made-it-happen/).

## Introduction
Current Meteor apps (v1.4.x, v1.5.x) can be run with Node.js v6.x. The pkg
repository offers multiple versions. This tutorial will explain how to install
Node 6 to get your app to run.

### Disclaimer / warning
As of writing this, Node.js just published
[a security report](https://nodejs.org/en/blog/vulnerability/july-2017-security-releases/).
This may happen again later. Make sure to be running a stable and patched
version on your production servers.

## Prerequisites on the server
### Install Node 6 and npm 3
As of writing this, the current stable version of Node is 8.1.3. However, Meteor
apps require at maximum Node 6 (the current LTS version) in order to build some
native modules, including `fibers`. So you will need to install the specific
versions:
```bash
pkg install node6 npm3
```

### Potential issues
In case you already hat the latest stable version of Node installed, pkg will
tell you about the conflict and offer to replace it, for example:
```bash
The following 3 package(s) will be affected (of 0 checked):

Installed packages to be REMOVED:
	node-8.1.3

New packages to be INSTALLED:
	node6: 6.11.0
	npm3: 3.10.10_2

Number of packages to be removed: 1
Number of packages to be installed: 2

The process will require 7 MiB more space.

Proceed with this action? [y/N]:
```

FreeBSD currently does not allow for multiple versions of Node to be installed
in parallel like you may know it from Python. Make sure that you have no other
apps installed that would require a more recent version of Node so they will not
break. Then you can enter `y` to continue.

## Deploying the application
### Create a tarball
In the project directory where you usually run `meteor`, run:
```bash
npm install         # to install dependencies from package.json
meteor build .      # to create the Meteor app bundle
```

This will create a file named `src.tar.gz` in the current directory.

### Upload the tarball to the server
```bash
ssh-add ~/.ssh/my-ssh-key-to-my-server
sftp freebsd@my-domain.org
put src.tar.gz
```

### Extract the tarball and install application dependencies
```bash
ssh freebsd@my-domain.org
tar -xf /home/freebsd/src.tar.gz             # extract
(cd bundle/programs/server && npm install)   # install node modules
```
The `README` in `bundle/` holds brief information on the procedure. You might
wish to look into it:
```bash
cat bundle/README
```

From here on, the app could already be run by simply invoking:
```bash
node bundle/main.js
```
This will give you an error that certain environment variables are not set, but
this way you can now confirm everything was installed correctly and your
application can be run - at least on the server side. The gist about the
environment variables is found in the aforementioned README file. For details,
please read the official Meteor documentation.

### Configuring reverse proxy (nginx)

#### TODO:

### Running as a system service

#### Preparation
As a first step, move the app to the global web server directory. This is my
personal preference to keep web-based apps in one place, but you can put it
anywhere else as you wish. You would just need to adjust the path in the rc
script.
```bash
sudo chown -R www:www bundle                 # set file permissions
sudo mv bundle /usr/local/www/my_app         # move to web server home
```

Then create directories where log and pid files will be stored. Log files will
help you understand what happened when errors occur or something just didn't
work, and the pid files are necessary for the service.
```bash
sudo mkdir -p /var/log/node /var/run/node    # create dirs for log and pid files
```

#### Creating a service
##### Init script
[FreeBSD rc scripts](https://www.freebsd.org/doc/en/articles/rc-scripting/)
reside in `/usr/local/etc/rc.d/`. This is where we will create our scripts as
well.
```bash
cd /usr/local/etc/rc.d/
```

With this tutorial comes a file `my_app` which can be used as a skeleton.

Hint: App names should not contain `-` because that would breake the name of the
variable in the script. This is why I chose `my_app` in this case. Otherwise you
will need to work around this and adjust the skeleton accordingly.

##### Configuration
Write the environment variables (env vars) into `/etc/rc.conf.d/my_app`:

```bash
my_app_enable=YES                                            # enable for auto-run on server startup and service start instead of onestart
NODE_ENV=production                                          # this is our production environment, check out the Node.js docs for more
PORT=3123                                                    # make sure to use one port per app, and remember it for the reverse proxy
ROOT_URL=https://my-domain.org/my_app                        # the root URL as seen from outside, used by Meteor.absoluteUrl
MONGO_URL=mongodb://mongouser:mongopw@127.0.0.1:27017/my_app # find another tutorial if you need to set up MongoDB ;)
```
