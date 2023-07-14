---
title: "TouchID ITerm2"
date: 2020-11-14T23:56:22+08:00
draft: false
---
```
sudo vim /etc/pam.d/sudo
//add the following code to the top of the file
auth sufficient pam_tid.so
```