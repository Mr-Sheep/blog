---
title: "使用 Lethe 來安全的清空硬碟數據"
date: 2022-07-27T18:05:57+08:00
draft: false
tags: ["Disk","macOS"]
categories: ["macOS"]
---

使用 [Lethe](https://github.com/Kostassoid/lethe) 來安全的擦除硬碟數據

<!--more-->

# Intro

[Lethe](https://github.com/Kostassoid/lethe)是一個跨平臺的，安全的，免費且開源的drive wiping utility.

其主要特點有：
- 支援 Windows，macOS 和 Linux。
- 支持數據校驗 （reads back）
- 使用快速的加密隨機發生器（fast cryptographic random generator）
- 自定義Block size
- 實驗性功能：追蹤並跳過壞掉的扇區
- 比直接使用`dd`命令更快

限制：
- 對於SSD來說，不可能可靠地擦除所有的數據，因為現代SSD控制器進行了各種優化，即磨損均衡和壓縮。目前最好的方法是使用隨機數據進行多輪擦除。之後可能會增加對安全擦除ATA命令的支持，使這個過程更加可靠。
- 每個存儲設備的最大塊數是2³²，即4,294,967,296。即當block size = 1MB時，可支援 4096TB。
- 該應用程序未在 RAID 存儲設備上進行測試

# 效能：
我實際測試下來一塊Western Digital 2TB MyPassport：
```bash
❯ sudo lethe wipe --verify=last -s=dod /dev/rdisk5
Wiping:
    Device              /dev/rdisk5
    Size                1.82TB (2000365289472 bytes)
    Scheme              DoD 5220.22-M / CSEC ITSG-06 / NAVSO P-5239-26, 3 passes
                        - fill with 0x00
                        - fill with 0xFF
                        - random fill
    Block size          1.00MB
    Starting offset     0B (0 bytes)
    Verification        Last stage only
Are you sure? (type 'yes' to confirm): yes

Stage 1/3: Performing Value Fill (00)
✔ Completed in 6 hours

Stage 2/3: Performing Value Fill (ff)
✔ Completed in 6 hours

Stage 3/3: Performing Random Fill
✔ Completed in 6 hours

Stage 3/3: Verifying Random Fill
✔ Completed in 6 hours
✔ Total time: 1 day
    Total device size       1.82TB
    Total device blocks     1907697
    Total blocks wiped      1907697
    Skipped blocks          0 (0%)
```

來自lethe作者的測試
- 測試環境：Macbook Pro 2015 with macOS 10.14.4 (Mojave) 
- 測試對象：Sandisk 64G Flash Drive with USB 3.0 interface. OS recommended block size is 128k.

**Zero fill**

 Command | Block size | Time taken (seconds)
---------|------------|----------
 `dd if=/dev/zero of=/dev/rdisk3 bs=131072` | 128k | 2667.21
 `lethe wipe --scheme=zero --blocksize=128k --verify=no /dev/rdisk3` | 128k | 2725.77
 `dd if=/dev/zero of=/dev/rdisk3 bs=1m` | 1m | 2134.99
 `lethe wipe --scheme=zero --blocksize=1m --verify=no /dev/rdisk3` | 1m | 2129.61

**Random fill**

 Command | Block size | Time taken (seconds)
---------|------------|----------
 `dd if=/dev/urandom of=/dev/rdisk3 bs=131072` | 128k | 4546.48
 `lethe wipe --scheme=random --blocksize=128k --verify=no /dev/rdisk3` | 128k | 2758.11

# 安裝 Lethe

推薦通過 [lethe/releases/latest](https://github.com/Kostassoid/lethe/releases/latest) 下載最新的lethe，可以將其放置於PATH中

# 使用

```bash
❯ lethe help
Lethe 0.7.0
https://github.com/Kostassoid/lethe
Secure disk wipe

USAGE:
    lethe <SUBCOMMAND>

OPTIONS:
    -h, --help       Prints help information
    -V, --version    Prints version information

SUBCOMMANDS:
    help    Prints this message or the help of the given subcommand(s)
    list    list available storage devices
    wipe    Wipe storage device
```

```bash
❯ lethe help wipe
lethe-wipe
Wipe storage device

USAGE:
    lethe wipe [FLAGS] [OPTIONS] <device>

FLAGS:
    -h, --help    Prints help information
    -y, --yes     Automatically confirm

OPTIONS:
    -b, --blocksize <blocksize>    Block size [default: 1m]
    -o, --offset <offset>          Starting offset (in bytes) [default: 0]
    -r, --retries <retries>        Maximum number of retries [default: 8]
    -s, --scheme <scheme>          Data sanitization scheme [default: random2x]  [possible values: badblocks, dod, gost,
                                   random, random2x, vsitr, zero]
    -v, --verify <verify>          Verify after completion [default: last]  [possible values: no, last, all]

ARGS:
    <device>    Storage device ID

Data sanitization schemes:
    badblocks     Inspired by a badblocks tool -w action., 4 passes
                  - fill with 0xAA
                  - fill with 0x55
                  - fill with 0xFF
                  - fill with 0x00
    dod           DoD 5220.22-M / CSEC ITSG-06 / NAVSO P-5239-26, 3 passes
                  - fill with 0x00
                  - fill with 0xFF
                  - random fill
    gost          GOST R 50739-95 (fake), 2 passes
                  - fill with 0x00
                  - random fill
    random        Single random fill, 1 pass
                  - random fill
    random2x      Double random fill, 2 passes
                  - random fill
                  - random fill
    vsitr         VSITR / RCMP TSSIT OPS-II, 7 passes
                  - fill with 0x00
                  - fill with 0xFF
                  - fill with 0x00
                  - fill with 0xFF
                  - fill with 0x00
                  - fill with 0xFF
                  - random fill
    zero          Single zeroes fill, 1 pass
                  - fill with 0x00

```

對於不同的scheme，可以閱讀：
- [Data Sanitization Methods](https://www.lifewire.com/data-sanitization-methods-2626133)
- [What is the difference between the different wiping methods used by DBAN?](https://security.stackexchange.com/questions/117724/what-is-the-difference-between-the-different-wiping-methods-used-by-dban)
- [DoD 5220.22-M Data Wipe Method](https://www.datadestroyers.eu/technology/dod_5220.22-m_data_wipe_method.html)
- [RCMP TSSIT OPS-II Data Wipe Method](https://www.datadestroyers.eu/technology/rcmp_tssit_ops-2.html)