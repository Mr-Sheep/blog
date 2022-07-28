---
title: "使用 Surge 的 Information Panel 編寫出口ip檢測腳本"
date: 2022-02-05T03:07:55+08:00
draft: false
tags: ["Surge"]
categories: ["Tweaks"]
---


如何使用Surge的Information Panel寫出出口ip檢測腳本？

<!--more-->
**Happy Lunar New Year 新年快樂，虎年順遂**

Surge iOS 從 4.9.3 版本開始就支援了 [information panel](https://community.nssurge.com/d/656-surge-ios-info-panel) 這個功能，個人一直覺得是比較陽春的一個功能，實際能夠使用到的場景並不多，裝完13就沒用了。

# 动态模式原理（[複製自surge論壇](https://community.nssurge.com/d/656-surge-ios-info-panel/)）：
可通过脚本控制面板内容。

```
[Panel]
PanelB = title="Panel Title",content="Panel Content\nSecondLine",style=info,script-name=panel

[Script]
panel = script-path=panel.js,type=generic
```

这个版本同时引入了 generic 类型脚本，该类型脚本为泛用类型，可被多种功能所调用。

当用户点击刷新按钮时，脚本将被唤起，传入参数为
```js
$input : {
    purpose: "panel",
    position: "policy-selection",
    panelName: "PanelB"
}
```
脚本应在 `$done()` 中返回 `title`、`content` 和 `style` 字段。

在脚本被第一次运行前，面板内容为定义行中的静态内容，运行后 Surge 会自动缓存上一次脚本的返回结果，并在执行刷新前始终显示上一次的脚本结果。

脚本样例：
```js
$httpClient.get("https://api.my-ip.io/ip", function(error, response, data){
	$done({
		title: "External IP Address",
		content: data,
	});
});
```

更多自定义:
- 当不传入 style 字段时，卡片将不显示图标，仅显示文字
- 当不传入 style 字段时，可传入 icon 字段，自定义图标，内容为任意有效的 SF Symbol Name，如 bolt.horizontal.circle.fill
- 当使用 icon 字段时，可传入 icon-color 字段控制图标颜色，字段内容为颜色的 HEX 编码。

# 現有腳本的問題

在GitHub上逛了一圈發現大多數腳本（如[這個](https://raw.githubusercontent.com/fishingworld/something/main/PanelScripts/net_info.js)）都是基於 [ip-api](http://ip-api.com) 或者是 [ipwhois](https://ipwhois.io/) 這類對於免費用戶僅支持http的API並且僅僅支持IPv4。作爲BGP Player ~~(糊路由的)~~，必須要來折騰一下。



# 從 0 開始成爲腳本小子

目前我找到的最合適的API是 [ip.sb](https://ip.sb/api/), 他們提供的接口支持免註冊獲取 IP Geo info，同時支援僅IPv6/IPv4請求 （[https://api-ipv4.ip.sb/ip](https://api-ipv4.ip.sb/geoip) 和 [https://api-ipv6.ip.sb/ip) ](https://api-ipv6.ip.sb/geoip)

選好API以後就可以來看看返回的json格式了：

```json
{
    "ip": "185.255.55.55",
    "country_code": "NL",
    "country": "Netherlands",
    "continent_code": "EU",
    "latitude": 52.3824,
    "longitude": 4.8995,
    "asn": "3214",
    "organization": "xTom Limited",
    "timezone": "Europe/Amsterdam",
}
```

我想要達成的效果是顯示IP地址和對應的ASN信息以及國家定位，地球icon會根據IP所在的大洲改變（🌍️🌎️🌏️）。我們需要 `ip`, `country`, `country_code`, `continent_code`, `asn` 和 `organization` 這五個值

```js
let url = "https://api-ipv4.ip.sb/geoip"
//let url = "https://api-ipv6.ip.sb/geoip"

$httpClient.get(url, function(error, response, data){
    let jsonData = JSON.parse(data)
    let ip = jsonData.ip
    let country = jsonData.country
    let emoji = getFlagEmoji(jsonData.country_code)
    let asn = jsonData.asn
    let asOrg = jsonData.asn_organization
    let continent = jsonData.continent_code
    const icon = {
      'AF': "globe.europe.africa.fill",
      'AN': "globe",
      'AS': "globe.asia.australia.fill",
      'EU': "globe.europe.africa.fill",
      'NA': "globe.americas.fill",
      'OC': "globe.asia.australia.fill",
      'SA': "globe.americas.fill",
      'default': "globe",
    };

  body = {
    title: "IPv4 Info",
    content: `IPv4: ${ip}\nAS${asn} ${asOrg}\n${emoji} ${country}`,
    icon: icon[continent] || icon["default"]
  }
  $done(body);
});



function getFlagEmoji(countryCode) {
    const codePoints = countryCode
      .toUpperCase()
      .split('')
      .map(char =>  127397 + char.charCodeAt());
    return String.fromCodePoint(...codePoints);
}
```

之後將其上傳到github（或者任何一個方便你更新的地方，並在surge配置中指向腳本地址即可）：
```js
[Panel]
IPv4 Network Info= script-name=IPv4 Network Info, title="IPv4 Network Info", content="Refresh", style=info, update-interval=60
IPv6 Network Info= script-name=IPv6 Network Info, title="IPv6 Network Info", content="Refresh", style=info, update-interval=60

[Script]
IPv4 Network Info= type=generic,timeout=3,script-path=https://raw.githubusercontent.com/Mr-Sheep/Random-Rules/master/Surge/Script/ipcheck-v4.js
IPv6 Network Info= type=generic,timeout=3,script-path=https://raw.githubusercontent.com/Mr-Sheep/Random-Rules/master/Surge/Script/ipcheck-v6.js
```