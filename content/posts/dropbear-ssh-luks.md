---
title: "unlock LUKS with Dropbear SSH public key"
date: 2022-07-31T14:38:32+08:00
draft: false
categories: ['ssh','luks']
tags: ['Linux']
cover:
    image: "/img/cover/Drop-Bear-Size-Chart.webp"
    caption: "image from https://mythicaustralia.com/drop-bear-size-chart/"
---

How to unlock LUKS enabled disk on headless linux machine

<!--more-->


![N5105 PCB](/img/n5105.webp)

I recently purchased this puppy and planned to use it as my home lab + router. I encrypted the entire disk with LUKS[^1] and wondered how to unlock it headlessly.

[Dropbear SSH](https://matt.ucc.asn.au/dropbear/dropbear.html) seems to be the optimal solution in this case. (Cuz we all love Aussie hoax :)))

# About Dropbear

Dropbear's webpage doesn't really work atm, but luckily we had [Dropbear SSH on archive.org](https://web.archive.org/web/20220712180156/https://matt.ucc.asn.au/dropbear/dropbear.html): 

>  Dropbear is a relatively small SSH server and client. It runs on a variety of unix platforms. Dropbear is open source software, distributed under a MIT-style license. Dropbear is particularly useful for "embedded"-type Linux (or other Unix) systems, such as wireless routers.

**Features**
> - A small memory footprint suitable for memory-constrained environments – Dropbear can compile to a 110kB statically linked binary with uClibc on x86 (only minimal options selected)
> - Dropbear server implements X11 forwarding, and authentication-agent forwarding for OpenSSH clients
> - Can run from inetd or standalone
> - Compatible with OpenSSH ~/.ssh/authorized_keys public key authentication
> - The server, client, keygen, and key converter can be compiled into a single binary (like busybox)
> - Features can easily be disabled when compiling to save space
> - Multi-hop mode uses SSH TCP forwarding to tunnel through multiple SSH hosts in a single command. dbclient user1@hop1,user2@hop2,destination


# Setup
The setup process is pretty straight forward. 

1. my setup: 
```
❯ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
NAME                    SIZE TYPE  FSTYPE      MOUNTPOINT
mmcblk0               116.5G disk
├─mmcblk0p1             512M part  vfat        /boot/efi
├─mmcblk0p2             488M part  ext2        /boot
└─mmcblk0p3           115.5G part  crypto_LUKS
  └─mmcblk0p3_crypt   115.5G crypt LVM2_member
    ├─lab--vg-root   114.5G lvm   ext4        /
    └─lab--vg-swap_1   976M lvm   swap        [SWAP]
mmcblk0boot0              4M disk
mmcblk0boot1              4M disk
```

2. to install dropbear SSH:
```
apt udpate && apt -y install dropbear-initramfs
```

3. to configure dropbear SSH with public key authentication, edit `/etc/dropbear-initramfs/config`, edit the `DROPBEAR_OPTIONS` field:
```
DROPBEAR_OPTIONS="-I 240 -p <your-port> -s -g -j -k"
```

{{< collapse summary="all options (click to expand)" >}}

```
❯ dropbear -h
Dropbear server v2020.81 https://matt.ucc.asn.au/dropbear/dropbear.html
Usage: dropbear [options]
-b bannerfile	Display the contents of bannerfile before user login
		(default: none)
-r keyfile      Specify hostkeys (repeatable)
		defaults:
		- dss /etc/dropbear/dropbear_dss_host_key
		- rsa /etc/dropbear/dropbear_rsa_host_key
		- ecdsa /etc/dropbear/dropbear_ecdsa_host_key
		- ed25519 /etc/dropbear/dropbear_ed25519_host_key
-R		Create hostkeys as required
-F		Don't fork into background
-E		Log to stderr rather than syslog
-m		Don't display the motd on login
-w		Disallow root logins
-G		Restrict logins to members of specified group
-s		Disable password logins
-g		Disable password logins for root
-B		Allow blank password logins
-T		Maximum authentication tries (default 10)
-j		Disable local port forwarding
-k		Disable remote port forwarding
-a		Allow connections to forwarded ports from any host
-c command	Force executed command
-p [address:]port
		Listen on specified tcp port (and optionally address),
		up to 10 can be specified
		(default port is 22 if none specified)
-P PidFile	Create pid file PidFile
		(default /var/run/dropbear.pid)
-i		Start for inetd
-W <receive_window_buffer> (default 24576, larger may be faster, max 1MB)
-K <keepalive>  (0 is never, default 0, in seconds)
-I <idle_timeout>  (0 is never, default 0, in seconds)
-V    Version
```
{{< /collapse >}}

4. place your public key into `/etc/dropbear-initramfs/authorized_keys` and then run `update-initramfs -u` to update the existing [initramfs](https://www.unix.com/man-page/Linux/8/update-initramfs/).


5. go ahead and reboot your system. you should now able to ssh into your system using `ssh root@<ip> -p <port> -i /path/to/private/key`. Then use the command `cryptroot-unlock` to unlock the disk.
	![dropbear operation](/img/dropbear-operation.webp)

Hope you noticed that this is my first post with a cover image, cuz, again, we all love Aussie hoax. Don't we?

# References
1. [How to unlock LUKS using Dropbear SSH keys remotely in Linux](https://www.cyberciti.biz/security/how-to-unlock-luks-using-dropbear-ssh-keys-remotely-in-linux/)



[^1]: https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup
