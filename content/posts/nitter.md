---
title: "Deploy Nitter with caddy/nginx and docker"
date: 2021-10-29T22:07:04+08:00
draft: false
tags: ["Twitter"]
categories: ["Privacy"]
---
A free and open source alternative Twitter front-end focused on privacy. 
<!--more-->

# Why?
This part is copied from: [Nitter's README.md](https://github.com/zedeus/nitter)

![Screenshot](https://raw.githubusercontent.com/zedeus/nitter/master/screenshot.png)

It's basically impossible to use Twitter without JavaScript enabled. If you try, you're redirected to the legacy mobile version which is awful both functionally and aesthetically. For privacy-minded folks, preventing JavaScript analytics and potential IP-based tracking is important, but apart from using the legacy mobile version and a VPN, it's impossible. This is is especially relevant now that Twitter removed the ability for users to control whether their data gets sent to advertisers.

Using an instance of Nitter (hosted on a VPS for example), you can browse Twitter without JavaScript while retaining your privacy. In addition to respecting your privacy, Nitter is on average around 15 times lighter than Twitter, and in most cases serves pages faster (eg. timelines load 2-4x faster).

In the future a simple account system will be added that lets you follow Twitter users, allowing you to have a clean chronological timeline without needing a Twitter account.

## Why own instance?
You can use a public nitter instance from [nitter/wiki/Instances](https://github.com/zedeus/nitter/wiki/Instances), or you can use my nitter instances:
```
is-nitter.resolv.ee // hosted in 🇮🇸 with Cloudflare CDN enabled
lu-nitter.resolv.ee // hosted in 🇱🇺 with Cloudflare Magic Transit
```
But if you don't want to trust other people, you can, and you should set up your own instance.
# Installation
## Docker
This blog post is based on [Debian 11](https://docs.docker.com/engine/install/debian/), if you are using other distros, refer to [Install Docker Engine from docker docs](https://docs.docker.com/engine/install/)

1. To install Docker Engine:
```
// Remove previously installed, if any, Docker
$ apt -y remove docker docker-engine docker.io containerd runc && apt update
$ apt -y install ca-certificates curl gnupg lsb-release

// Setup repo
$ curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
$(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
  
// Install Docker
$ apt update
$ apt -y install docker-ce docker-ce-cli containerd.io

// Start Docker
$ systemctl start docker
$ systemctl enable docker
```

2. Create a `nitter.conf` in the current word directory, or you can download it from [https://github.com/zedeus/nitter/blob/master/nitter.conf](https://github.com/zedeus/nitter/blob/master/nitter.conf)
```
[Server]
address = "0.0.0.0"
port = 8080
https = false  # disable to enable cookies when not using https
httpMaxConnections = 100
staticDir = "./public"
title = "nitter"
hostname = "nitter.net"

[Cache]
listMinutes = 240  # how long to cache list info (not the tweets, so keep it high)
rssMinutes = 10  # how long to cache rss queries
redisHost = "localhost"
redisPort = 6379
redisConnections = 20 # connection pool size
redisMaxConnections = 30
redisPassword = ""
# max, new connections are opened when none are available, but if the pool size
# goes above this, they're closed when released. don't worry about this unless
# you receive tons of requests per second

[Config]
hmacKey = "secretkey" # random key for cryptographic signing of video urls
base64Media = false # use base64 encoding for proxied media urls
tokenCount = 10
# minimum amount of usable tokens. tokens are used to authorize API requests,
# but they expire after ~1 hour, and have a limit of 187 requests.
# the limit gets reset every 15 minutes, and the pool is filled up so there's
# always at least $tokenCount usable tokens. again, only increase this if
# you receive major bursts all the time

# Change default preferences here, see src/prefs_impl.nim for a complete list
[Preferences]
theme = "Nitter"
replaceTwitter = "nitter.net"
replaceYouTube = "piped.kavin.rocks"
replaceInstagram = ""
proxyVideos = true
hlsPlayback = false
infiniteScroll = false
```

3. To run prebuilt Nitter in Docker:
```
$ docker run -v $(pwd)/nitter.conf:/src/nitter.conf -d -p 8080:8080 zedeus/nitter:latest

// change 8080 to the port you want
```

## Caddy
If you want a better performance, you should consider using nginx.

As recommended by nitter, one should run caddy behind a reverse proxy for security reason; in this case, I'll use [caddy](https://github.com/Caddyserver/caddy)

1. To install caddy:
Download the latest release from [here](https://github.com/caddyserver/caddy/releases/latest)

To install the latest release for Debian:
```bash
$ curl -s https://api.github.com/repos/caddyserver/caddy/releases/latest | grep "browser_download_url.*linux_amd64.deb" | cut -d '"' -f 4 | wget -i - | dpkg -i -
```

Caddyfile sample config:
```
:443, your-domain.com
tls no@body.me
log {
  level debug
}
route {
  reverse_proxy 127.0.0.1:8080
}
```

After that start caddy with systemd:
```
$ systemctl restart caddy
$ systemctl enable caddy
```
Head over to `your-domain.com` to enjoy your nitter instance.

## NGINX
### Installing nginx (copied from [nginx.org](https://nginx.org/en/linux_packages.html#Debian))

    sudo apt install curl gnupg2 ca-certificates lsb-release debian-archive-keyring

Import an official nginx signing key so apt could verify the packages authenticity. Fetch the key:

    curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
        | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

Verify that the downloaded file contains the proper key:

    gpg --dry-run --quiet --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg

The output should contain the full fingerprint 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 as follows:

    pub   rsa2048 2011-08-19 [SC] [expires: 2024-06-14]
          573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
    uid                      nginx signing key <signing-key@nginx.com>

If the fingerprint is different, remove the file.

To set up the apt repository for stable nginx packages, run the following command:

    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    http://nginx.org/packages/debian `lsb_release -cs` nginx" \
        | sudo tee /etc/apt/sources.list.d/nginx.list

To install nginx, run the following commands:

    sudo apt update
    sudo apt install nginx -y

You also need `certbot` or `acme.sh` installed to issue certificates; simply run `sudo apt -y install certbot` and `certbot certonly --standalone -d <your-domain>`. You can also use Cloudflare's Origin CA if your nitter domain is proxied by Cloudflare.

# keep nitter up-to-date
Create a new shell script called `what-ever-you-like.sh`
```bash
containerId=$(docker ps -q --filter ancestor=zedeus/nitter)
docker pull zedeus/nitter:latest
docker stop $containerId
docker rm $containerId
docker run -v $(pwd)/nitter.conf:/src/nitter.conf -d --network host zedeus/nitter:latest 
```

`crontab -e` to run the script every day at 6 am.