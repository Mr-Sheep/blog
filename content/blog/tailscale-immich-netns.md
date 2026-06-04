+++
title = "Tailscale alongside with WireGuard using netns"
date = "2026-06-04T04:59:49-05:00"

description = "how to get tailscale and wireguard vpn to work nicely together"

tags = ["fun","randomstuff","ssh","system","linux","self-hosting"]
+++

I have previously setup my homelab in [this post](/homelab-finally)

There's one problem: I want to use a VPN service to proxy all my outgoing traffics except those from tailscale since the its performance sucks with [immich](https://immich.app/).

I tried to use `ip route` to filter out some prefixes (including derps) but it did not work well.  So after reading [this post](https://jamesguthrie.ch/blog/multi-tailnet-unlocking-access-to-multiple-tailscale-networks/) from the internet, I have this stupid but working idea.

I also wanted everything to be automatically setup during boottime, so most things below will be in shell scripts (and thanks gpt). And for extra clarity, here's how the final topology looks like: 

```
Tailscale client
   ↓
tailns: tailscale0
   ↓
tailns veth: 10.200.0.2
   ↓
root veth: 10.200.0.1
   ↓
nginx listening on 443
```

## Creating a netns

I used the following script to create the netns, add corresponding nics to the netns, add a veth pair so that we can talk to the root namesapce in `/usr/local/sbin/setup-tailns.sh`: 

```sh
#!/bin/sh
set -eu

NS=tailns
WAN=wan0

ROOT_VETH=veth-root
NS_VETH=veth-tail

ROOT_IP=10.200.0.1/30
NS_IP=10.200.0.2/30

# Create namespace if missing
ip netns list | grep -q "^${NS}" || ip netns add "$NS"

# Bring loopback up
ip netns exec "$NS" ip link set lo up

# Move WAN NIC into namespace if still in root namespace
if ip link show "$WAN" >/dev/null 2>&1; then
    ip link set "$WAN" down
    ip link set "$WAN" netns "$NS"
fi

# Bring WAN up inside namespace
if ip netns exec "$NS" ip link show "$WAN" >/dev/null 2>&1; then
    ip netns exec "$NS" ip link set "$WAN" up
fi

# Create veth pair if missing
if ! ip link show "$ROOT_VETH" >/dev/null 2>&1 && \
   ! ip netns exec "$NS" ip link show "$NS_VETH" >/dev/null 2>&1; then
    ip link add "$ROOT_VETH" type veth peer name "$NS_VETH"
    ip link set "$NS_VETH" netns "$NS"
fi

# Configure root side veth
ip addr flush dev "$ROOT_VETH" || true
ip addr add "$ROOT_IP" dev "$ROOT_VETH"
ip link set "$ROOT_VETH" up

# Configure namespace side veth
ip netns exec "$NS" ip addr flush dev "$NS_VETH" || true
ip netns exec "$NS" ip addr add "$NS_IP" dev "$NS_VETH"
ip netns exec "$NS" ip link set "$NS_VETH" up
```

and it's systemd service `/etc/systemd/system/tailns-setup.service`: 

```systemd
[Unit]
Description=setup tailns netns
After=systemd-udev-settle.service
Wants=systemd-udev-settle.service
Before=tailscaled-tailns.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/setup-tailns.sh

[Install]
WantedBy=multi-user.target
```

don't worry about `tailscaled-tailns.service`, it will be in the next step :)

## Setting up local interfaces

Put your nic configs in `/etc/netns/tailns/network/interfaces`, and then create `/etc/systemd/system/tailns-wan.service`: 

```systemd
[Unit]
Description=configure wan0
Requires=tailns-setup.service
After=tailns-setup.service
Before=tailscaled-tailns.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/ip netns exec tailns /usr/sbin/ifup --force -i /etc/netns/tailns/network/interfaces wan0
ExecStop=/usr/sbin/ip netns exec tailns /usr/sbin/ifdown --force -i /etc/netns/tailns/network/interfaces wan0

[Install]
WantedBy=multi-user.target
```

## Setting up tailscale

We can't use `tailscale up` anymore since we want the daemon in the `tailns` namespace:

`/etc/systemd/system/tailscaled-tailns.service`:

```systemd
[Unit]
Description=tailscaled daemon
Requires=tailns-setup.service tailns-wan.service
After=tailns-setup.service tailns-wan.service
Wants=network-online.target

[Service]
Type=simple
RuntimeDirectory=tstail
StateDirectory=tstail
ExecStart=/usr/sbin/ip netns exec tailns /usr/sbin/tailscaled \
  --tun=tailscale0 \
  --socket=/run/tstail/tstail.socket \
  --statedir=/var/lib/tstail \
  --state=/var/lib/tstail/tstail.state
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

## Forwarding Traffic using socat

`/etc/systemd/system/tailns-socat-https.service`:

```systemd
[Unit]
Description=proxy tailscale https traffic
Requires=tailscaled-tailns.service
After=tailscaled-tailns.service

[Service]
Type=simple
ExecStart=/usr/sbin/ip netns exec tailns /usr/bin/socat \
  TCP-LISTEN:443,bind=<your-tailscale-ip>,fork,reuseaddr \
  TCP:10.200.0.1:443
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

## Firewall

I just need ssh and https from tailscale to be accepted, `/usr/local/sbin/setup-tailns-firewall.sh`： 

```sh
#!/bin/sh
set -eu

IPT="ip netns exec tailns iptables"
IP6T="ip netns exec tailns ip6tables"

$IPT -P INPUT DROP
$IPT -P FORWARD DROP
$IPT -P OUTPUT ACCEPT

$IPT -F INPUT
$IPT -F FORWARD

$IPT -A INPUT -i lo -j ACCEPT
$IPT -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
$IPT -A INPUT -i tailscale0 -p tcp --dport 22 -j ACCEPT
$IPT -A INPUT -i tailscale0 -p tcp --dport 443 -j ACCEPT

$IP6T -P INPUT DROP
$IP6T -P FORWARD DROP
$IP6T -P OUTPUT ACCEPT

$IP6T -F INPUT
$IP6T -F FORWARD

$IP6T -A INPUT -i lo -j ACCEPT
$IP6T -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
$IP6T -A INPUT -i tailscale0 -p tcp --dport 22 -j ACCEPT
$IP6T -A INPUT -i tailscale0 -p tcp --dport 443 -j ACCEPT
```

and again: 

`/etc/systemd/system/tailns-firewall.service`:

```systemd
[Unit]
Description=tailns firewall rules
Requires=tailns-setup.service
After=tailns-setup.service
Before=tailns-socat-https.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/setup-tailns-firewall.sh

[Install]
WantedBy=multi-user.target
```

## Finally

time to enable all of them, gl:

```sh
systemctl daemon-reload
systemctl enable tailns-setup.service tailns-wan.service tailscaled-tailns.service tailns-socat-https.service tailns-firewall.service
```


{{< giscus "Mr-Sheep/blog" "MDEwOlJlcG9zaXRvcnkzNDQ4NjQ1MTQ=" "preferred_color_scheme" >}}