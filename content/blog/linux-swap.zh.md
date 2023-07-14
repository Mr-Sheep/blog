---
title: "Linux Swap"
date: 2020-04-05T02:17:21+08:00
draft: false
---

有些虛擬伺服器的記憶體給的很少，這個時候就需要虛擬記憶體了

<!--more-->

創建虛擬記憶體的步驟如下，默認在root下執行

- 創建Swapfile, 啓用，並更改權限

  ```
  dd if=/dev/zero of=/swapfile bs=1M count=你想要的大小
  mkswap /swapfile
  swapon /swapfile
  chmod 600 /swapfile
  ```

- 開機啓動,編輯 `/etc/fstab`

  ```
  /swapfile swap swap defaults 0 0
  ```

- 關閉虛擬記憶體

  ```swapoff -v /swapfile
  swapoff -v /swapfile
  ```

