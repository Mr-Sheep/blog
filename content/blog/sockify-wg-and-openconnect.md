+++
title = "socksify wireguard and openconnect"
date = "2023-12-11T19:08:28-06:00"
tags = ["vpn", "shadowsocks", "wireguard", "openconnect"]

+++

I always have multiple proxy connections from one devices, however, multiple wireguard tunnel can eat your battery. I've tried solutions like [gluten](https://github.com/qdm12/gluetun), but having to run a separate container for each wireguard connection almost always results in high ram/cpu usage.

With shadowsocks's recently added `outbound_bind_interface` support, I settled on the current solution: [shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust) + wireguard.

## SSServer

Download the latest version of shadowsocks-rust from [github](https://github.com/shadowsocks/shadowsocks-rust/releases/latest).

Extract the package, and move it to `/usr/local/bin/ssserver` (or where ever you want). Since we are going to run multiple instance of ss-server, we can use `systemd` to demonize it:

```
[Unit]
    Description=Shadowsocks-Rust Custom Server Service for %I
    Documentation=man:ss-server(1)
    After=network-online.target
    Wants=network-online.target

[Service]
    Type=simple
    CapabilityBoundingSet=CAP_NET_BIND_SERVICE
    AmbientCapabilities=CAP_NET_BIND_SERVICE
    DynamicUser=true
    ExecStart=/usr/local/bin/ssserver -c /etc/shadowsocks/%i.json

[Install]
    WantedBy=multi-user.target
```

For the actual configuration file, with shadowsocks-rust, you are no longer able put multiple server address in a list/array, instead you need a separate object for IPv4 and IPv6:

```json
{
  "servers": [
    {
      "server": "0.0.0.0",
      "server_port": 8388,
      "password": "<your-password-here>",
      "method": "chacha20-ietf-poly1305"
    },
    {
      "server": "::",
      "server_port": 8388,
      "password": "<your-password-here>",
      "method": "chacha20-ietf-poly1305"
    }
  ],
  "outbound_bind_interface": "wg0",
  "mode": "tcp_and_udp"
}
```

set the `outbound_bind_interface` to the corresponding interface.

## Wireguard

Because wireguard, by default, adds routes to the table, and that's something we don't want. I added `Table = off` to the wireguard config, and also `PersistentKeepalive` to keep my connection active.

```
[Interface]
  Table = off
  PrivateKey = <private-key>
  Address = <internal-ipv4>/32, <internal-ipv6>/128

[Peer]
  PublicKey = <public-key>
  AllowedIPs = 0.0.0.0/0,::0/0
  Endpoint = [<ipv6-endpoing>]:3017
  PersistentKeepalive = 25
```

after you are done with the wireguard config, `wg-quick` to bring it up and `systemctl enable --now ssserver@wg0` to start the ssserver for this tunnel.

## openconnect

~~some university is obsessed with tls based vpns.~~

to run openconnect in the background, we can simply add the `-b` flag, but I chose to run it inside a `screen`:

```
/usr/bin/sh -c 'echo "<passowrd>" | /usr/sbin/openconnect -u <username> vpn.<something>.edu --authgroup=5 --passwd-on-stdin --reconnect-timeout=60'
```

notice the `authgroup` here is optional, check with your school documentation to get the correct auth group.

To prevent the server from terminating the connection due to inactivity, we can have another screen that sends icmp through the tunnel:

```
screen ping -i 60 <something>.edu
```

The only problem with this solution is that openconnect will add some routes to the system routing table preventing you from using the shadowsocks connection when you are on campus. A possible solution is to run shadowsocks and [openconnect inside a namespace](https://austinjadams.com/blog/running-select-applications-through-anyconnect/)

then connect the ns with veth:

```
ip link add veth type veth peer name dummy1
ip link set veth netns <ns-name>
ip netns exec <ns-name> ip addr add 192.168.10.2/24 dev veth
ip netns exec <ns-name> ip link set veth up
ip addr add 192.168.10.1/24 dev dummy1
ip link set dummy1 up
```

and port forward traffic to shadowsocks:

```
iptables -t nat -A PREROUTING -d <your-public-ip>/32 -p udp --dport 8388 -j DNAT --to-destination 192.168.10.2:8388
iptables -t nat -A PREROUTING -d <your-public-ip>/32 -p tcp --dport 8388 -j DNAT --to-destination 192.168.10.2:8388
```
