+++
title = "macOS, UTM, WeChat"
date = "2025-10-02T19:11:01-05:00"

tags = ["fun","macos", "wechat"]
+++

我就是不想在 host 上安裝微信怎麼辦

{{< figure src="/blog/macos-utm-wechat/utm-wechat.webp" class="center-fig">}}

<!--more-->

今天剛好突發奇想，想看看 linux 版本的微信是否已經到了能夠正常使用的狀態

雖然 MAS 版本的 WeChat 是有原生的沙盒機制的，但是我就是看他不爽，多年來也從沒有在筆電上安裝過。

{{< figure src="/blog/macos-utm-wechat/macos-wechat.webp" class="center-fig">}}

很湊巧的是今天剛好出了最新的 [4.1.0.10 test](https://linux.weixin.qq.com/en)（官網並沒有 changelog 是通過[這篇 v2ex 帖子](https://www.v2ex.com/t/1163189)知道的

根據 v2ex 帖子，4.1.0.10 有如下更新：

	- 图片 OCR 选择文字功能移植过来了；

	- UI 全部改成大圆角了；

	- 有未读消息时，任务栏图标能闪动了；

	- 朋友圈界面可以发送朋友圈，而不是只能看了。长按相机按钮也可以发送纯文字，和手机逻辑一致；

	- 整体界面流畅度相比上个版本有巨大提升，尤其是内置浏览器和切换聊天窗口的卡顿大大缩短。

本來是想嘗試用 orbstack 安裝的，但是 orbstack 沒有原生的 GUI 界面，需要通過 X11 / VNC 的方式實現，不是很想折騰（微信並不是我的必需品），然後就想起了 UTM

## UTM 

使用任意虛擬化軟體即可，用 UTM 是因爲他輕量，開源，免費

可以在 utm [官網的 gallery 裡](https://mac.getutm.app/gallery/)挑選你喜歡的 linux 發行版（或者是自己從頭配置）。官方提供的鏡像會自動將網路等全都配置好可以開箱即用。

UTM 官網提供的鏡像需要在 Internet Archive 下載會被要求登錄。幸運的是有熱心鄉民提供了 mirror：[https://mirror.bouwhuis.network/utm/](https://mirror.bouwhuis.network/utm/) 

我選擇的是 Debian 11 (Xfce)，標註體積僅 882.8MB 

```
默認用戶/密碼：debian/debain
root用戶密碼：root/password
```

## WeChat

要是能夠直接一鍵 `dpkg -i` 就可以完成安裝也就不用單獨寫一個 section 了：

1. 在安裝微信之前你需要先安裝依賴字體：

	```sh
	sudo apt -y update
	sudo apt -y install fonts-noto-cjk
	```

2. 你還需要以下：

	```sh
	sudo apt -y install libatomic1 libxkbcommon-x11-0 libxcb-icccm4 libxcb-image0 libxcb-render-util0 libxcb-keysyms1
	```

3. 安裝完成以後就可以正常安裝並使用微信了：

	```sh
	wget https://dldir1v6.qq.com/weixin/Universal/Linux/WeChatLinux_arm64.deb 
	
	sudo dpkg -i WeChatLinux_arm64.deb
	```
	

聊天產生的文件位於 `/home/debian/Documents/xwechat_files`.

{{< giscus "Mr-Sheep/blog" "MDEwOlJlcG9zaXRvcnkzNDQ4NjQ1MTQ=" "preferred_color_scheme" >}}
