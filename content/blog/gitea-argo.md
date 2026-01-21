+++
title = "Self-hosting Gitea with Cloudflare Tunnel for ssh"
date = "2026-01-21T16:26:51-06:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["self-hosting", "gitea", "cloudflare"]
+++

A simple walkthru of how I self host my gitea instance with nginx and cloudflare zero access to enable ssh.

<!--more-->

## Part 0: setting up gitea

### Gitea binary
I do not want to host it using docker, so I went thru the binary installation route. Gitea already have a [pretty decent how-to guide](https://docs.gitea.com/installation/install-from-binary) on their site, this part is mainly a copy-paste from their doc.

At the time of writing this blog post, the release version used was `1.25.3`, and the server used was running debian 13:

```sh
wget -O gitea https://dl.gitea.com/gitea/1.25.3/gitea-1.25.3-linux-amd64
chmod +x gitea
mv gitea /usr/local/bin/gitea
```

1. creating a dedicated user for gitea to be run as:

    ```bash
    adduser \
        --system \
        --shell /bin/bash \
        --gecos 'Git Version Control' \
        --group \
        --disabled-password \
        --home /home/git \
        git
    ```

2. creating the directories:

    ```bash
    mkdir -p /var/lib/gitea/{custom,data,log}
    chown -R git:git /var/lib/gitea/
    chmod -R 750 /var/lib/gitea/
    mkdir /etc/gitea
    chown root:git /etc/gitea
    chmod 770 /etc/gitea
    ```

3. to run `gitea` using systemd, we can download the sample `gitea.service` from the gitea repo:

    ```sh
    wget -O /etc/systemd/system/gitea.service https://github.com/go-gitea/gitea/raw/refs/heads/release/v1.25/contrib/systemd/gitea.service
    systemctl daemon-reload
    systemctl enable gitea --now
    ```

### DB setup

per gitea's official documentation:

> Be aware that SQLite does not scale; if you expect your instance to grow at a later time, you should choose another database type.

to future proof our instance, I chose to use [postgresql 18](https://www.postgresql.org):

```sh
apt install -y postgresql-common
/usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
apt install postgresql-18
```

login to the db using `su -c "psql" - postgres`, and create `gitea` role and `giteadb` database:

```postgresql
CREATE ROLE gitea WITH LOGIN PASSWORD '<your-password>';

CREATE DATABASE giteadb WITH OWNER gitea TEMPLATE template0 ENCODING UTF8 LC_COLLATE 'en_US.UTF-8' LC_CTYPE 'en_US.UTF-8';
```

you will also need to modify the `pg_hba.conf` located at `/etc/postgresql/18/main/pg_hba.conf`:

```sh
local   giteadb         gitea                                   scram-sha-256
```

you can test your db connection with `psql -U gitea -d giteadb`

> from the doc: Rules on `pg_hba.conf` are evaluated sequentially, that is the first matching rule will be used for authentication. Your PostgreSQL installation may come with generic authentication rules that match all users and databases. You may need to place the rules presented here above such generic rules if it is the case.

### Reverse Proxying with Nginx

here's my nginx config:

```nginx
server {
  listen 443 ssl;
  listen [::]:443 ssl;
  http2 on;
  ssl_certificate       <path-to-cert>;
  ssl_certificate_key   <path-to-privkey>;
  ssl_protocols         TLSv1.2 TLSv1.3;
  ssl_ciphers           ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_session_tickets   off;
  ssl_stapling          off;
  ssl_stapling_verify   off;
  server_name           <your-server-name>;

  location / {
    client_max_body_size 512M;
    proxy_pass http://localhost:3000;
    proxy_set_header Connection $http_connection;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

now you can go to yoursite and start the gitea setup, we will be changing the ssh domain for gitea later. The config file is located at `/etc/gitea/app.ini`. The list of configurable options can be found [here](https://docs.gitea.com/administration/config-cheat-sheet).

## Part 1: setting up cloudflared

the web interface and https based git are proxied by cloudflare, but we will need argo tunnels for ssh based accesses.

we will mainly be following this [guide](https://developers.cloudflare.com/cloudflare-one/tutorials/gitlab/) from cloudflare.

### Creating an application

we will need to create an application on Cloudflare ZeroTrust, go to `Access Controls` -> `Applications`, and add an application there with the application hostname you intended to use and access policies. we can keep everything else default.

### Getting the latest cloudflared binary

1. download deb

    ```sh
    wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
    dpkg -i cloudflared-linux-amd64.deb
    ```

    setting up systemd for it

    ```sh
    cloudflared service install
    ```

2. log in to your cf account and create a new tunnel, it will assign a uuid that we need later:

    ```sh
    cloudflared login

    cloudflared tunnel create gitea
    ```

3. configure the tunnel `vim ~/.cloudflared/config.yml`:

    ```yaml
    tunnel: <uuid>
    credentials-file: /root/.cloudflared/<uuid>.json

    ingress:
    - hostname: git-ssh.example.com
        service: ssh://localhost:22
    # Catch-all rule, which just responds with 404 if traffic doesn't match any of
    # the earlier rules
    - service: http_status:404
    ```

    valid your tunnel config using `cloudflared tunnel ingress validate` and bring it up with `systemctl start cloudflared` (or `cloudflared tunnel run`)

### Creating DNS records

add a cname record pointing to `<uuid>.cfargotunnel.com` with the hostname you intended to use.

### Updating the gitea's `app.ini`

under `[server]`, update `SSH_DOMAIN` to the hostname you intend to use, then restart gitea using `systemctl restart gitea`.

## Part 2: Client setup

1. upload your ssh public key to the gitea instance in `example.com/user/settings/keys`.
2. follow [the instructions above](#getting-the-latest-cloudflared-binary) to get the latest `cloudflared` binary 
3. append `~/.ssh/config` with the following:
    ```sh
    Host git-ssh.example.com
        ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
    ```
4. attempt to git clone a new repo would prompt you to authenticate

    ```sh
    git clone git@git-ssh.example.com:test/test
    ```