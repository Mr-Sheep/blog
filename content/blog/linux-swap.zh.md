---
title: "Linux Swap"
date: 2019-04-05T02:17:21+08:00
draft: false
---

note to remind myself how to setup swap on linux

<!--more-->

創建虛擬記憶體的步驟如下，默認在 root 下執行

- 創建 Swapfile, 啓用，並更改權限

  ```sh
  dd if=/dev/zero of=/swapfile bs=1M count=你想要的大小
  mkswap /swapfile
  swapon /swapfile
  chmod 600 /swapfile
  ```

- 開機啓動,編輯 `/etc/fstab`

  ```sh
  /swapfile swap swap defaults 0 0
  ```

- 關閉虛擬記憶體

  ```sh
  swapoff -v /swapfile
  swapoff -v /swapfile
  ```
