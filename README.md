## Introduction

Meteor works with legacy Node.js (v0.10.x). The pkg repository, however, offers the latest version.

### Research
Let's consult Google. We end up in (https://forums.freebsd.org/threads/52004/)[the FreeBSD forums].

So there is (http://www.freshports.org/www/npm)[a port for npm] (a description how to build a package).

### Preparation
First things first. We need to upgrade the ports tree, so let's (https://www.freebsd.org/doc/handbook/ports-using.html)[RTFM] how to do that again.

Assuming we already had a tree:
```bash
portsnap fetch update
```

### Configuration
First cd to the npm port:
```bash
cd /usr/ports/www/npm
```

And then use `NODE010` as mentioned in the forum. (https://www.freebsd.org/cgi/man.cgi?ports(7))[The man page says] `config` lets us change the OPTIONS. When running
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
