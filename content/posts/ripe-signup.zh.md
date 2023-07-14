---
title: "RIPE NCC 註冊以及相關字段填寫"
date: 2021-09-11T19:33:29+08:00
draft: false
ShowToc: true
tags: ["BGP"]
categories: ["networking"]
---

拿到自己的第一個ASN快2年了，來重新寫下某個去世的人曾經的博文吧，避免各種教程上不更新到way back machine的地址


<!--more-->

# RIPE 介紹
[RIPE NCC](https://www.ripe.net/)(Réseaux IP Européens Network Coordination Centre)是全球五大RIR（Regional Internet Registries）之一，提供網路資源分配，註冊和協調服務。

![Map of service regions of the regional Internet registries](https://upload.wikimedia.org/wikipedia/en/d/d5/RIR_Service_Regions_Map_2015.jpg.png)
## Why RIPE NCC
- 支持個人和公司申請
- LIR價格： (€2,000 setup + €700 semi-annually) * (1 + 21% VAT) = €3,267.00
- 面板陽間

# 註冊 RIPE NCC 帳號以及字段創建

## 註冊帳號

打開RIPE的註冊頁面並填入相關信息。郵箱請務必真實且長期使用，免得帶來不必要的麻煩。郵箱可以更換，但是不可以更換後不可以使用之前的地址。

註冊🔗️：[https://access.ripe.net/registration](https://access.ripe.net/registration)

## 字段創建
前往：[Select object type you would like to create](https://apps.db.ripe.net/db-web-ui/webupdates/select) 創建相關字段

2.1 role and maintainer pair

![Screen Shot 2021-09-11 at 7.49.53 PM.png](https://i.loli.net/2021/09/11/Fv4dAKamcrzfsXJ.png)

**mntner**: 維護者的字段名，純英文，可包含"-"和"_"，建議格式為*-MNT，例如MrSheep-MNT

**role**: 維護者的稱呼，可以寫全名或其他名稱。可包含英文、數字和.`'_-

**address**: 地址

**e-mail**: 手機號，建議格式為+國家區號.手機號，例如+86.19260817666

創建完成之後會得到`role`和`mnter`兩個字段；`role`可以用於`abuse-c`和`tech-c`,`admin-c`等字段；`mnter`可以用於`mnt-by`，`mnt-ref`和`mnter`等字段

2.2 organization

同樣前往：[Select object type you would like to create](https://apps.db.ripe.net/db-web-ui/webupdates/select) 創建相關字段

![RIPEorg.png](https://i.loli.net/2021/09/11/hEqenFm7uBvP2dI.png)

`org-type`: 默認即可

`address`: 地址欄，如果你要申請ASN請填寫證件國家對應的地址；公司請填寫公司註冊地址

`e-mail`: 郵箱

`abuse-c`: 之前創建的`role`

`mnt-ref`: 之前創建的`mnter`

參考：
[BGP入门 - RIPE账号注册与必要字段创建](https://web.archive.org/web/20190227042014/https://www.ykhut.com/archives/72/)

BGP交流群：[MoeQing Zoo](https://t.me/MoeQing)