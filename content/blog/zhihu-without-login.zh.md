---
slug: "zhihu-no-login"
title: "通過 Surge 和 1Blocker 實現手機上免登錄使用知乎網頁版"
date: 2020-10-31T14:26:32+08:00
draft: false
tags: ["Zhihu"]
categories: ["Tweaks"]
---

如何幹掉知乎彈出的登錄窗口？

<!--more-->

如果直接打開 Google/duckduckgo 出來的知乎結果，只能查看一個回答，會被強制跳轉到 `oia.zhihu.com`

由於 desktop 上的知乎不會被強制跳轉：

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

可以通過使用 [Surge](https://www.nssurge.com/) 修改`User-Agent`達到強制使用桌面版知乎的效果：

```yaml
// 移除原有UA
^https?://(www|api).zhihu.com/question header-del User-Agent

// 重新添加新的UA
^https?://(www|api).zhihu.com/question header-add User-Agent "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_5 (Ergänzendes Update)) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.1.1 Safari/605.1.15"
```

UA 可以根據需要隨意修改，可以在[UA DB](https://udger.com/resources/ua-list)挑選.

後來我找到了 uBlockOrigin 的解決方案：[Add zhihu.com modal login popup #8314](https://github.com/uBlockOrigin/uAssets/pull/8314)。在 iOS safari 上實現可以用到[1Bocker](https://1blocker.com/)，直接添加`block url`規則，block 掉：`static.zhihu.com/heifetz/main.signflow.*.js`即可
