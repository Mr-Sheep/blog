---
title: "使用 Knot DNS 設置 IPv6 反向解析"
date: 2022-06-26T18:05:57+08:00
draft: false
tags: ["DNS", "IPv6","Linux"]
categories: ["networking"]
---

使用 [Knot DNS](https://www.knot-dns.cz/) 爲自己的前綴建立反向解析並啓用[dnssec](https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en)

<!--more-->

# Intro

[Knot DNS](https://www.knot-dns.cz/)是一個高性能的權威域名伺服器(high-performance authoritative-only DNS)，支援現代域名系統的所有關鍵功能。相似的軟件有[BIND](https://www.isc.org/bind/)，[PowerDNS](https://www.powerdns.com/)等。

Knot DNS 使用 C 和 LuaJIT 編寫，包含一個解析器和守護進程，的主要特點有：
- 開源 （Open Source）
- 功能豐富（Feature Packed）
- 高性能（High Performance）
- 安全穩定（Secure and Stable）
- 個人覺得Knot DNS的module也是一個非常好的feature。

The Knot Re­solver is a caching full re­solver im­ple­men­ta­tion writ­ten in C and LuaJIT, in­clud­ing both a re­solver li­brary and a dae­mon. Mod­u­lar ar­chi­tec­ture of the li­brary keeps the core tiny and ef­fi­cien­t, and pro­vides a state-­ma­chine-­like API.

本文基於Knot DNS [3.1.8](https://www.knot-dns.cz/2022-04-28-version-318.html)編寫。如果對Knot DNS的感興趣可以閱讀[官方文檔](https://www.knot-dns.cz/docs/latest/html/)。

# Knot DNS 的安裝

推薦通過 [https://www.knot-dns.cz/download/](https://www.knot-dns.cz/download/) 下載最新的Current Stable Branch，選擇對應的distro即可。

Debian：
```bash
#!/bin/bash

## Enable this repository:

unset SUDO
if [ "$(whoami)" != "root" ]; then
    SUDO=sudo
fi

${SUDO} apt-get -y install apt-transport-https lsb-release ca-certificates wget
${SUDO} wget -O /usr/share/keyrings/knot.gpg https://deb.knot-dns.cz/apt.gpg
${SUDO} sh -c 'echo "deb [signed-by=/usr/share/keyrings/knot.gpg] https://deb.knot-dns.cz/knot-latest/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/knot-latest.list'
${SUDO} apt-get update

## Install Knot DNS server:

${SUDO} apt-get install knot

## Install Knot DNS utilities:

${SUDO} apt-get install knot-dnsutils

```

啓動：`systemctl start knot`
重啓：`systemctl restart knot`
停止：`systemctl stop knot`


# Knot DNS 配置

### [Base](https://www.knot-dns.cz/docs/latest/html/configuration.html#zone-templates)

Knot只需要很短的幾行配置就可以跑起來：
```
# This is a sample of a minimal configuration file for Knot DNS.
# See knot.conf(5) or refer to the server documentation.

server:
    rundir: "/run/knot"
    user: knot:knot
    # automatic-acl: on
    listen: [ 0.0.0.0@53, ::@53 ]

log:
  - target: syslog
    any: info

database:
    storage: "/var/lib/knot"
```



### [Zone templates](https://www.knot-dns.cz/docs/latest/html/configuration.html#zone-templates)

模板就是模板，不需要解釋。模板之間不存在繼承關係。
```
template:
  - id: default
    storage: "/var/lib/knot/"
    file: "%s.zone"
    
  - id: slave
    storage: "/var/lib/knot/slave"
    file: "%s.zone"
```

### [Access Control List（ACL）](https://www.knot-dns.cz/docs/latest/html/configuration.html#access-control-list-acl) 

`automatic-acl`如果未啓用， 所有需要授權的請求都會被拒絕。可以通過添加`acl`區塊來限制授權的請求。

對於master來說，需要授權slave伺服器進行`zone transfer`:
```
acl:
  - id: acl_slave
    address: <slave-ip>
    action: transfer
```

對於slave，需要允許master進行`notify`:
```
acl:
  - id: acl_master
    address: <master-ip>
    action: notify
```

如果感興趣secondary（slave）是如何工作的：[How Does Secondary DNS Work?](https://blog.cloudflare.com/secondary-dns-deep-dive/)

關於TSIG的配置請參考：[Secondary (slave) zone](https://www.knot-dns.cz/docs/latest/html/configuration.html#secondary-slave-zone)

### remote
master:
```
remote:
  - id: secondary
    address: <slave-ip>@<port>
```

slave (secondary):
```
remote:
  - id: master
    address: <master-ip>@<port>
```

### DNSSEC

DNSSEC通過向現有的DNS記錄添加加密簽名來創建一個安全的域名系統。這些數字簽名與A、AAAA、MX、CNAME等常見記錄類型一起存儲在DNS名稱服務器中。通過檢查其相關的簽名，可以驗證請求的DNS記錄來自其權威的名稱服務器，並且在途中沒有被改變，而不是在中間人攻擊中注入的假記錄。如果感興趣dnssec如何運作：[How DNSSEC Works](https://www.cloudflare.com/dns/dnssec/how-dnssec-works/)

選擇Knot DNS的另外一大原因則是其支援自動簽發DNSSEC，用戶只需要在`master`定義一個`policy`即可:
```
policy:
  - id: default
    algorithm: ecdsap384sha384
    ksk-lifetime: 365d
    zsk-lifetime: 30d
    nsec3: on
```

`algorithm`可選參考：[Domain Name System Security (DNSSEC) Algorithm Numbers](https://www.iana.org/assignments/dns-sec-alg-numbers/dns-sec-alg-numbers.xhtml)。推薦使用`ECDSAP256SHA256`或`ECDSAP256SHA256`。使用ECDSA達到128-bit security僅需256-bit，RSA則需要3072bit。

關於ECDSA和RSA的選擇推薦閱讀：[ECDSA: The missing piece of DNSSEC](https://www.cloudflare.com/dns/dnssec/ecdsa-and-dnssec)

### [mod-synthrecord](https://www.knot-dns.cz/docs/latest/html/modules.html#synthrecord-automatic-forward-reverse-records)

這個插件可以自動爲prefix創建格式爲`2a0c-2222-30--1.<origin>.`反向解析記錄，可以通過手動添加覆蓋:
```
mod-synthrecord:
  - id: <unique-id>
    type: reverse
    origin: <後綴，如snorlax.blue>
    network: 2a0c:2222::/32
````



### zone
以`2a0c:2222::/32`爲例：

master：
```
zone:
  - domain: 2.2.2.2.c.0.a.2.ip6.arpa
    notify: secondary
    acl: acl_slave
    module: mod-synthrecord/<unique-id>
    dnssec-signing: on
    dnssec-policy: default
```

slave:
```
  - domain: 2.2.2.2.c.0.a.2.ip6.arpa
    template: slave
    master: master
    acl: acl_master
```

### 完整配置示例
{{< collapse summary="master">}}

```

server:
    rundir: "/run/knot"
    user: knot:knot
    automatic-acl: on
    listen: [ 0.0.0.0@53, ::@53 ]

log:
  - target: syslog
    any: info

database:
    storage: "/var/lib/knot"

policy:
  - id: default
    algorithm: ecdsap384sha384
    ksk-lifetime: 365d
    zsk-lifetime: 30d
    nsec3: on

remote:
  # secondary slave server
  - id: secondary
    address: <slave-ip>@<slave-port>

acl:
  - id: acl_slave
    address: <slave-ip>
    action: transfer

template:
  - id: default
    storage: "/var/lib/knot/"
    file: "%s.zone"

mod-synthrecord:
  - id: <unique-id>
    type: reverse
    origin: 
    network: 2a0c:2222::/32
    
zone:
    # Primary zone
  - domain: 2.2.2.2.c.0.a.2.ip6.arpa
    notify: secondary
    acl: acl_slave
    module: mod-synthrecord/<unique-id>
    dnssec-signing: on
    dnssec-policy: default
```

{{< /collapse >}}


{{< collapse summary="slave" >}}

```
server:
    rundir: "/run/knot"
    user: knot:knot
    automatic-acl: on
    listen: [ 0.0.0.0@53, ::@53 ]

log:
  - target: syslog
    any: info

database:
    storage: "/var/lib/knot"

remote:
  # secondary slave server
  - id: master
    address: <master-ip>@<master-port>

acl:
  - id: acl_master
    address: <master-ip>
    action: notify

template:
  - id: slave
    storage: "/var/lib/knot/slave"
    file: "%s.zone"

zone:
    # slave zone
  - domain: 2.2.2.2.c.0.a.2.ip6.arpa
    template: slave
    master: master
    acl: acl_master
```
{{< /collapse >}}


### [Knotc](https://www.knot-dns.cz/docs/latest/html/man_knotc.html#knotc-knot-dns-control-utility)

和`bird`提供的`birdc`類似，`knot`提供了`knotc`。

日常需要的一般只有4條，更多action和options請參考：https://www.knot-dns.cz/docs/latest/html/man_knotc.html#options 和 https://www.knot-dns.cz/docs/latest/html/man_knotc.html#actions

```
knotc zone-begin zone... 
knotc zone-set zone owner [ttl] type rdata
knotc zone-unset zone owner [type [rdata]]
knotc zone-commit zone...
```


# RIPE DB domain object

使用[此連結](https://apps.db.ripe.net/db-web-ui/webupdates/wizard/RIPE/domain)或點擊「Create an Object」- 「Object Type：domain」

![domain object wizard, source: ripe](https://www.ripe.net/manage-ips-and-asns/db/support/rdnsdemo-1.png/@@images/image/large)

注意`nserver`字段不支持直接使用ip地址，且不可以解析到相同IP，需要至少兩個IPv4 nserver。如果你不想使用你自己的secondary server，可以使用`ns.ripe.net`

{{< collapse summary="具體要求">}}

> Keep in mind that, for a /16 (v4) and /32 (v6), you can use ns.ripe.net as the secondary server. In both cases, you have to allow zone transfers from the name server listed in the SOA resource record's MNAME field to the RIPE NCC distribution servers. The IP addresses of the two servers are:
> 
> 193.0.19.190 / 2001:67c:2e8:11::c100:13be
> 
> 93.175.159.250 / 2001:67c:2d7c:66::53
> 
> If your servers are configured to send DNS notify messages and you would like ns.ripe.net to update promptly, please send them to the IP addresses listed here. Any notify messages sent directly to the addresses of ns.ripe.net will not be seen. Also bear in mind that we do not support any non-standard configurations (such as port numbers other than 53, TSIG keys and so on).
{{< /collapse >}}

在點擊加號添加兩條`ds-rdata`，在master ns上運行`keymgr <zone-name> ds`，將DS後的`<Keytag> <Algorithm> <Digest type> <Digest>`貼入，submit通過檢測即可

# DNSSEC 校驗

1. https://dnssec-analyzer.verisignlabs.com/
2. https://dnsviz.net

![DNSSEC Authentication Chain](https://dnsviz.net/d/2.4.6.e.c.0.a.2.ip6.arpa/YrgtSw/dnssec/auth_graph.svg)

# Reference
- [RIPE NCC: Configuring Reverse DNS](https://www.ripe.net/manage-ips-and-asns/db/support/configuring-reverse-dns)
- [Knot DNS documentation](https://www.knot-dns.cz/docs/latest/html/)
- [Knot DNS 使用教程](https://blog.groverchou.com/2020/08/10/Knot-DNS-%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B/)