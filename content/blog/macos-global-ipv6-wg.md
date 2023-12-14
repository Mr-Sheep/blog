+++
slug = "macos-global-ipv6-wg"
title = "wireguard as IPv6 tunnelbroker on macOS"
date = "2023-12-11T18:30:09-06:00"
draft = false
description = "How to use wireguard as IPv6 tunnelbroker on macOS"

tags = ["macOS","wireguard","IPv6"]

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."
+++

Usually I don't use wireguard as a tunnelbroker on my mac, because A. it sucks B. I have [shadowsocks](https://shadowsocks.org) as a more power-efficient alternative.[^1]

For this post, the server I am using is running debian 12, the setup for the server side:

```conf
[Interface]
  PrivateKey = <$Private-Key>
  Address = <prefix>:1::/64
  MTU = 1420

[Peer]
  PublicKey = <$Public-Key>
  AllowedIPs = <prefix>:1::1/64
```

Now here's the tricky part, normally if I want to use wireguard as a tunnelbroker, all I need to do is set the `AllowedIPs` on the client side to be `::/0`. But macOS somehow decided to [change the IPv4 routes as well](https://www.reddit.com/r/WireGuard/comments/ewt7g0/comment/fnl6apb):

when `AllowedIPs` is set to `::/0`:

```sh
❯ netstat -rn | grep default
default            192.168.76.1       UGScIg            en1
```

notice the `I` [flag](https://www.reddit.com/r/WireGuard/comments/ewt7g0/ios_macos_route_all_traffic_through_peer_only_ipv6/) here, this route is associated with an interface scope

normally:

```bash
❯ netstat -rn | grep default
default            192.168.76.1       UGScg             en1
```

and from `route monitor`:

```bash
❯ route monitor
got message of size 164 on Tue Dec 12 08:46:54 2023
RTM_DELETE: Delete Route: len 164, pid: 94, seq 249, errno 0, flags:<GATEWAY,DONE,STATIC,PRCLONING,CONDEMNED,GLOBAL>
locks:  inits:
sockaddrs: <DST,GATEWAY,NETMASK,IFP,IFA>
 default 192.168.66.1 default en1:14.98.77.xx.xx.xx 192.168.66.80
```

We could manually add the default route back to the routing table again using `sudo route -n add default 192.168.66.1` since wireguard on macos does not support `PostUP` and `PostDown` commands.

A [redditer](https://www.reddit.com/r/WireGuard/comments/ewt7g0/comment/fnl897x) provided an way easier solution, by splitting up `::/0` into smaller prefixes, for example `::/1` and `8000::/1`. Anything larger than `/120` [should work](https://github.com/WireGuard/wireguard-apple/blob/2fec12a6e1f6e3460b6ee483aa00ad29cddadab1/Sources/WireGuardKit/PacketTunnelSettingsGenerator.swift#L137):

```conf
[Interface]
  PrivateKey = <Private Key>
  ListenPort = 51820
  Address = <prefix>:1::1/64
  DNS = 2606:4700:4700::1111
  MTU = 1420

[Peer]
  PublicKey = <Public Key>
  AllowedIPs = ::/1, 8000::/1
  Endpoint = <endpoint>:51820
  PersistentKeepalive = 25
```

[^1]: This is a bold claim as I never really tested it, but anyways
