---
title: "通過Surge和1Blocker實現手機上免登錄使用知乎網頁版"
date: 2021-10-31T14:26:32+08:00
draft: false
tags: ["Zhihu"]
categories: ["Tweaks"]
---

如何幹掉知乎彈出的登錄窗口？
<!--more-->

知乎是一個垃圾內容非常多的網站，你可以在上面找到各式各樣的極端言論，但是也有一些很多年前寫下的有用的內容。如果直接打開Google/Duckduckgo出來的知乎結果，只能查看一個默認答案，然偶會被強制跳轉到`oia.zhihu.com`


然後就想到了網頁版知乎不會被跳轉，隨便找任意知乎請求可知：
```post
GET /heifetz/favicon.ico HTTP/2
Host: static.zhihu.com
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 11_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/94.0.4606.81 Safari/537.36
Accept: image/avif,image/webp,*/*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Origin: https://www.zhihu.com
DNT: 1
Connection: keep-alive
Sec-Fetch-Dest: image
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-site
```


可以通過使用[Surge](https://www.nssurge.com/)魔改User-Agent達到強制使用桌面版知乎的效果：
```yaml
// 移除原有UA
^https?://(www|api).zhihu.com/question header-del User-Agent

// 重新添加新的UA
^https?://(www|api).zhihu.com/question header-add User-Agent "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_5 (Ergänzendes Update)) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.1 Safari/605.1.15"
```
UA我是隨便選的，如果想要定製自己的UA，可以隨便找一個[UA DB](https://udger.com/resources/ua-list)挑選.

這個時候打開知乎你會發現知乎會彈出一個登錄窗口，但是因爲屏幕尺寸太小，無法顯示出關閉按鈕。經過搜索，找到了uBlockOrigin的解決方案：[Add zhihu.com modal login popup #8314](https://github.com/uBlockOrigin/uAssets/pull/8314)。爲了更方便的直接在iOS safari上實現，會用到[1Bocker](https://1blocker.com/)這個神器了

直接添加`block url`規則，block掉：`static.zhihu.com/heifetz/main.signflow.*.js`即可

忙瘋了水一下文章QQ