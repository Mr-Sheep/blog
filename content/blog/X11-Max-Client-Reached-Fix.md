---
slug: "max-client-reached"
title: "X: Max Client Reached Fix"
date: 2020-08-30T21:12:08+08:00
draft: false
description: "X11 Maximum Client Fix"
resources:
tags: ["X11", "Linux"]

lightgallery: false
---

A quick fix for `X: Maximum number of client reached`

<!--more-->

When trying to run some app on using X, I got this error message:

```shell
cannot open display: :0.0
Maximum number of clients reached
```

---

After some google searches, I found this as a "solution":

**edit your `xorg.conf` file, I'm using xpra, so `/etc/xpra/xorg.conf`.**

For other display managers, it may be located in `/etc/X11/` or somewhere else.
Find the `ServerFlags` section, add this to your file:

```
Option "MaxClients" "512"
```

To find out what value is supported for MaxClient, simply run `Xorg` with a very large `-maxclient` flag

```
/usr/lib/xorg/Xorg -maxclients 1000000000

------

(EE)
Fatal server error:
(EE) maxclients must be one of 64, 128, 256, 512, 1024 or 2048
(EE)
(EE)
Please consult the The X.Org Foundation support
	 at http://wiki.x.org
 for help.
(EE)


```

### Credit

[Does X-Window have a maximum number limit on clients?](https://unix.stackexchange.com/questions/498652/does-x-window-have-a-maximum-number-limit-on-clients)

[How can I diagnose/debug “maximum number of clients reached” X errors?](https://askubuntu.com/questions/4499/how-can-i-diagnose-debug-maximum-number-of-clients-reached-x-errors)
