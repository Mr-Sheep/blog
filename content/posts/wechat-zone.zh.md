---
title: "關於微信轉區的一些迷思"
date: 2021-09-06T00:19:16+08:00
draft: false
ShowToc: true
tags: ["WeChat"]
categories: []

---

微信最近開放了區域互轉。允許用戶從Wexin轉移到WeChat。 因爲實在無法忍受微信朋友圈的廣告，準備轉移到歐洲區試試

<!--more-->

# 如何轉移？

將「微信」綁定的電話號碼更改爲非+86電話號碼即可。

以下內容來自[Telegram @DocOfCard](https://t.me/DocOfCard/1590):

- 转移过程TOS加载缓慢可能出现短暂空白
  - 必须是在填入验证码之前弹协议才算转移成功，如果没弹协议直接到了输入验证码步骤就失败了，直接放弃
  - 转完后几分钟内自动登出，提示数据转移完成(如图)，重新登录后确认为EEA帐户，有CallKit（iOS ）
  - 如果弹协议但是一直没有自动登出并要求重新登录，需要换个其他国家号码开VPN再转一次
  - 搜索微信团队，查看账号主体在哪[区别](https://t.me/DocOfCard/1592)
- 由于众所周知的原因，转移前建议先备份
- 已经开通的健康码不影响
- 可能影响注册需要人脸验证的小程序，852/853帐户可以试试注册的时候添加额外86号码
- 非86帐户无法使用网页微信，电脑客户端微信也风控严格，可能遇到登录成功后帐户被锁需要自助(好友)解封

## 不同地區之間帳號的區別

[Weixin/WeChat各区账号功能对比](https://go.docofcard.com/2HzWdH)

（如果加載不出來這個Telegram Post說明你梯子掛了）

{{< telegramPost DocOfCard 1592 >}}

# Fun Facts

## 數據保存服務器

在成功更換了微信的地區以後，本來以爲只是名字變了，結果真的是遷移了所有的數據。

通過Surge查看微信發出去的請求，聊天請求被發往了

```
IP-CIDR,101.32.0.0/16,no-resolve
IP-CIDR,129.226.0.0/16,no-resolve
```

經過查詢以後這兩個段都是騰訊雲的prefix，均位於[AS132203](https://bgp.he.net/AS132203#_prefixes)下（和騰訊雲海外業務位於同樣的ASN下）且真實位於新加坡。與其他業務位於一個機房內：

![WechatIMG2344.jpeg](https://i.loli.net/2021/09/06/YyzHEcvU9RBFIhk.jpg)



當嘗試使用「騰訊雲新加坡」訪問微信時，Surge會提示發往微信服務器的所有請求僅有上行，無下行

![Screen Shot 2021-09-06 at 1.04.30 AM.png](https://i.loli.net/2021/09/06/WZPajV9z4RTfiFI.png)

## 數據導出

![E8DEF059-EB5C-42DC-88E1-196B53FD2BFC.jpeg](https://i.loli.net/2021/09/06/sKrwQnyuCf3t9mB.jpg)

Coming soon （郵件一直沒有收到）

