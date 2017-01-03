---
layout: post
title: Foray into node.js on Windows 7
categories:
- Programming
tags:
- nodejs
- javascript
- express
date: 2014-05-20 00:00:00 +1000
---
I've decided to learn about [node.js](http://nodejs.org/) - it's about time, really.  I've used it - but only so much as to run a grunt task to bundle a specific revision of
an Angular directive.

I found [this neat tutorial](http://cwbuecheler.com/web/tutorials/2013/node-express-mongo/) for picking up some node.js know-how - looks pretty decent, and I'll give feedback
on that in a subsequent post.  Meanwhile, here's a rundown of getting the basic environment up for those on Windows who are having issues (as I did)

<!--break-->
Installing NodeJS with NPM
----------
I must have had this idea a little while ago, because I thought my first step should be to find out what I've got installed so far - it turns out I have
2 versions installed from [Chocolatey](https://chocolatey.org/), and one from the official installer, all old versions.  So I got rid of them all, and discovered
from the helpful comments at the [package page for nodejs](https://chocolatey.org/packages/nodejs) that I should use
[nodejs.install](https://chocolatey.org/packages/nodejs.install) instead.

    c:\Development\node-learning>node -v
    v0.10.28

    c:\Development\node-learning>npm -v
    1.4.9

Woohoo!

NPM issues
----------
Installing [express](http://expressjs.com/) proved difficult - the mime package kept giving me permission errors and this:

    npm ERR! Error: ENOENT, chmod 'C:\Users\*****\AppData\Roaming\npm\node_modules\express\node_modules\accepts\node_modules\mime\README.md'

Whatever that means.

I kept going around in circles - the prevailing wisdom seems to be to [clean the npm cache](http://alicoding.com/how-to-fix-error-enoent-lstat-npm-when-trying-to-install-modules/)
or to [run as administrator](http://stackoverflow.com/questions/15272542/socket-io-installation-fails-on-windows-7-32-bit/23569456#23569456).

I tried these (alongside an `npm uninst -g` for everything to get a clean start), and it worked a treat.  I then removed all the modules again, cleaned my cache,
and tried again, to make sure my solution was valid.

    npm ERR! EEXIST, mkdir 'C:\Users\*****\AppData\Roaming\npm\node_modules\express\node_modules\send\node_modules\mime'
    File exists: C:\Users\*****\AppData\Roaming\npm\node_modules\express\node_modules\send\node_modules\mime
    Move it away, and try again.

It wasn't. I've made sure `%appdata%/npm-cache` is empty as well, to no avail.  I sometimes get a few other errors, which seem to cycle around - even though the cache
and modules are empty

    npm ERR! Error: ENOENT, chmod 'C:\Users\*****\AppData\Roaming\npm\node_modules\express\node_modules\type-is\node_modules\mime\LICENSE'

    npm ERR! Error: ENOENT, lstat 'C:\Users\*****\AppData\Roaming\npm\node_modules\express\node_modules\send\node_modules\mime\types\mime.types'

I'm sure I got an `EPERM` here as well.

I've since learned that `ENOENT` means "No such file or directory".  I'm well into yak-shaving territory at this point.

The light!
----------
In the end, I just kept running `npm install express -g` until it worked. It's intermittent, which makes me think this stuff installs in parallel or there's a load balancer
serving the wrong file or something.  Not much of a light, but them's the breaks.

Running `npm install` for the package had similar issues.  As before the "keep on trying" Method works, and works in fewer iterations if you install
it package-at-a-time.  I couldn't find any documentation about parallel installs, so I've no idea what's going on.

Be sure to let me know if you know what's going on here!

In any case, it was easy sailing from here on out. The tutorial was straight forward,
though it reminded me of my PHP days (jam the DB object into the request object? _Really_?)
