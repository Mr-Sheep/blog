---
title: "Veracrypt Time Machine"
date: 2020-10-23T14:43:02+08:00
draft: false
tags: ["OS X","Time Machine","VeraCrypt"]
categories: ["system"]
---

I've got an external G-DOCK, a thunderbolt hard drive bay with VeraCrypt enabled. I want to use it for my Mac's Time Machine backups.  
<!--more-->
[According to Apple](https://support.apple.com/en-us/HT202784#format), Time Machine supports all Mac OS Extended (Journaled) formats and Xsan formats. In my case, the container is mounted exactly as Mac OS Extended(Journaled), but Time Machine **does not** allow me to select this volume.

To solve this problem, all we need to do is manually add the partition, fire up a terminal window and type:
```bash
sudo tmutil setdestination -a /Volume/NameOfYourVolume
```
You could check your Time Machine destinations with `tmutil destinationinfo`
```
❯ tmutil destinationinfo
====================================================
Name          : File Vault
Kind          : Local
Mount Point   : /Volumes/File Vault
ID            : F3ECB098-25B8-40E9-92D8-D39CF4C6AB6B
```

References: 
[Time Machine backups to a Veracrypt volume?](https://apple.stackexchange.com/questions/284450/time-machine-backups-to-a-veracrypt-volume)