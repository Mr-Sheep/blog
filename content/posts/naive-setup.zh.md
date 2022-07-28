---
title: "Naive 配置"
date: 2021-05-13T17:05:24+08:00
draft: false
categories: ['Naive','V2Ray']
tags: ['Censorship']
---

Make a fortunate quietly

<!--more-->

本文的内容你可以直接在[klzgrad/naiveproxy/wiki/简体中文](https://github.com/klzgrad/naiveproxy/wiki/%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)中閱讀到更完整的版本

# NaïveProxy ![build workflow](https://github.com/klzgrad/naiveproxy/actions/workflows/build.yml/badge.svg)

NaïveProxy使用Chrome的网络堆栈来伪装流量，具有很强的抗审查能力和低可探测性。重用Chrome的堆栈也确保了性能和安全方面的最佳实践。

NaïveProxy可以缓解以下几种流量攻击：

* 通过HTTP/2中的流量复用[缓解](https://arxiv.org/abs/1707.00641)指纹识别/流量分类
* 通过重复使用[Chrome的网络堆栈](https://arxiv.org/abs/1607.01639)缓解对于TLS参数指纹识别
* 通过「应用前置」，将代理服务器隐藏在一个常用的前端（Caddy）与应用层路由后面）来防御主动探测
* 通过填充长度缓解基于长度的流量分析。

NaïveProxy的代码由一系列补丁组成，基于每个新版本的Chrome进行变形和重构。

## 架构

[浏览器 → Naïve (客户端)] ⟶ 审查者 ⟶ [前端 → Naïve (服务端)] ⟶ Internet

NaïveProxy使用Chrome的网络栈。所以防火墙截获的流量行为与Chrome和标准前端服务器（如Caddy、HAProxy）之间的常规HTTP/2流量是完全相同的。

前端的Caddy会将未经过认证的用户和主动探测流量重定向至后端的伪装服务器（或其他网站），使其无法检测到代理的存在，例如：

[探测] ⟶ 前端 ⟶ [本地伪装index.html]

从v84版本开始，用户可以使用包含naïve的caddy forwardproxy分支来代替单独的naïve服务端

## 下载

最简单的跨平台方式是从[GitHub Releases](https://github.com/klzgrad/naiveproxy/releases/latest)下载编译好的二进制包

目前支援的平台有：Linux，Windows，Mac OS X以及OpenWRT

用户**应始终使用最新版本**，以保持签名与Chrome相同。

注意：在Linux上使用预编译好的二进制文件之前，必须先安装`libnss3`

## 配置

### 服务端

在服务器上, 用包含naïve的caddy forwardproxy分支编译并运行Caddy v2 **(Golang v1.14+)**

```sh
$ go get -u github.com/caddyserver/xcaddy/cmd/xcaddy
$ ~/go/bin/xcaddy build --with github.com/caddyserver/forwardproxy@caddy2=github.com/klzgrad/forwardproxy@naive
$ sudo setcap cap_net_bind_service=+ep ./caddy
```

创建`/etc/caddy/Caddyfile`，加入一下内容（替换 `<user> `& `<pass>`）

```
:443, example.com
tls me@example.com
route {
  forward_proxy {
    basic_auth <user> <pass>
    hide_ip
    hide_via
    probe_resistance
  }
  file_server { root /var/www/html }
}
```

`:443` 必须出现在文件头部，这个Caddyfile才可以正常工作。对于更高级的用法，可以考虑[使用JSON进行Caddy 2的配置](https://caddyserver.com/docs/json/)。

如果需要让caddy以守护进程的方式运行，请参考[Run Caddy as daemon](https://github.com/klzgrad/naiveproxy/wiki/Run-Caddy-as-a-daemon)

### 客户端

在本地运行`./naive`和以下`config.json`，使naïve监听本地`1080`的socks请求

`./naive config.json`

```json
{
  "listen": "socks://127.0.0.1:1080",
  "proxy": "https://user:pass@example.com"
}
```

参见[USAGE.txt](https://github.com/klzgrad/naiveproxy/blob/master/USAGE.txt)以了解`config.json`中的更多参数。另见[性能优化](https://github.com/klzgrad/naiveproxy/wiki/Performance-Tuning)。

### 和V2Ray 共存

```json
:443, example.com
tls me@example.com
log {
  level debug // 僅作爲故障排查用
}
route {
  forward_proxy {
    basic_auth <user> <pass>
    hide_ip
    hide_via
    probe_resistance
  }
  reverse_proxy /path/ 127.0.0.1:29708 {
                header_up Host {http.request.host}
                header_up X-Real-IP {http.request.remote.host}
                header_up X-Forwarded-For {http.request.remote.host}
                header_up X-Forwarded-Port {http.request.port}
                header_up X-Forwarded-Proto {http.request.scheme}
}
	redir https://fake.com
}
```

## 自行编译

如果你不喜欢下载二进制文件，你可以自行编译NaïveProxy。

要求:

* Ubuntu (apt-get install): git, python2, ninja-build (>= 1.7), pkg-config, libnss3-dev, curl, unzip, ccache (optional)
* MacOS (brew install): git, ninja, ccache (optional)
* Windows ([choco install](https://chocolatey.org/)): git, python2, ninja, visualstudio2017community. See [Chromium's page](https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md#Visual-Studio) for detail on Visual Studio setup requirements.

编译 (输出至 to `./out/Release/naive`):

```
$ git clone --depth 1 https://github.com/klzgrad/naiveproxy.git
$ cd naiveproxy/src
$ ./get-clang.sh
$ ./build.sh
```

改脚本会使用`curl`从Google服务器下载所需要的工具。你可能需要为curl设置一个代理环境变量，例如：`export ALL_PROXY=socks5h:/127.0.0.1:1080`。

## 

