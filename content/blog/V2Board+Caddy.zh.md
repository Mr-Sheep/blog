---
slug: "v2board-kickstart"
title: "如何成爲帶機場主：面板搭建"
date: 2020-11-12T20:56:10+08:00
draft: false
tags: ["v2board", "Anti-Censorship"]
categories: ["networking", "Proxy"]
---

如何使用 Nginx + PHP + mariadb 搭建自己的 v2board

<!--more-->

實際上 v2board 的前端寫的並不是很好，很容易就可以看到被隱藏的套餐。sspanel 也是一個不錯的選擇。

本文默認使用`root`用戶（不推薦你這麼做），請對如`composer`等使用單獨的賬戶

# Changelog

Jul 28th 2022: 移除 caddy 部分

Nov 15th 2021: 添加`php7.4-bcmath`

# PHP

1. 添加 Ondřej Surý 的 ppa:

```
wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" |  tee /etc/apt/sources.list.d/php.list
apt update
```

2. 安裝`php7.4`和`redis`

```
apt -y install php7.4 php7.4-fpm php7.4-common php7.4-cli redis php7.4-bcmath
以及一大堆亂七八糟的包 -->
apt -y install php7.4-curl php7.4-redis php7.4-mysql php7.4-mbstring php7.4-json php7.4-opcache php7.4-readline php7.4-xml

```

在安裝之後建議 hold php 的版本或註釋掉 repo

3. 解除被禁用的函數
   以`php7.4`爲例，編輯位於 `/etc/php/7.4/fpm/php.ini`
   找到

```php
disable_functions = "show_source, system, shell_exec, exec"
```

將`disable_functions`中的`putenv`,`proc_open`,`pcntl_alarm`,`pcntl_signal`从列表中删除

4. 修改配置

   編輯`/etc/php/7.4/fpm/php-fpm.conf`，在最後添加

   ```php
   include=/etc/php/7.4/fpm/www.conf
   ```

5. 編輯`/etc/php/7.4/fpm/pool.d/www.conf`，修改

   ```
   user = nginx
   group = nginx
   listen.owner = nginx
   listen.group = nginx
   listen.mode = 0660
   ```

   保證 nginx 能夠正常使用 fastcgi unix socket，需要一致的用戶

6. 啓動

   ```
   systemctl restart php7.4-fpm.service
   systemctl status php7.4-fpm.service
   ```

# MariaDB

```bash
curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash
```

使用`systemctl status mysql`驗證 mysql 是否成功安裝

```bash
● mariadb.service - MariaDB 10.6.8 database server
     Loaded: loaded (/lib/systemd/system/mariadb.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-06-25 22:49:38 PDT;
```

## 加固 MySQL

```
mysql_secure_installation
```

```
Securing the MySQL server deployment.

Connecting to MySQL using a blank password.

VALIDATE PASSWORD COMPONENT can be used to test passwords
and improve security. It checks the strength of password
and allows the users to set only those passwords which are
secure enough. Would you like to setup VALIDATE PASSWORD component?
對於沒有遠程訪問的數據庫，這個作爲可選項，對於遠程數據庫，建議開啓

Press y|Y for Yes, any other key for No:
按y或Y代表是，其餘按鍵爲否

設置數據庫root密碼
Please set the password for root here.

New password:

Re-enter new password:

移除匿名用戶
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.

禁用對於root賬戶的遠程訪問
Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : y
Success.

刪除test數據庫
By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.


Remove test database and access to it? (Press y|Y for Yes, any other key for No) : y
 - Dropping test database...
Success.

 - Removing privileges on test database...
Success.

重載權限
Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
Success.

All done!

```

## 如何移除密碼安全性驗證插件？

```
    mysql -h localhost -u root -p
    uninstall plugin validate_password;
    如果上面的命令不管用
    UNINSTALL COMPONENT 'file://component_validate_password';
```

## 創建數據庫

數據庫用戶爲`v2b`

用戶密碼爲`fuckGFW`

數據庫名字爲`v2board`

爲了保證數據庫安全，請不要給予用戶 `*.*`權限

**MySQL 5.X**

```sql
mysql -u root -p
輸入密碼
GRANT ALL PRIVILEGES ON v2board.* TO 'v2b'@'localhost' IDENTIFIED BY 'fuckGFW（密碼）';
flush privileges;

退出數據庫
\q

mysql -u v2b -p
輸入你剛剛創建好的密碼
CREATE DATABASE v2board;

退出數據庫
\q
```

**MySQL 8.X**

```sql
mysql> CREATE DATABASE v2board;
mysql> CREATE USER 'v2b'@'localhost' IDENTIFIED BY 'fuckGFW(你的密碼)';
mysql> GRANT ALL PRIVILEGES ON v2board.* TO 'v2b'@'localhost' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
mysql> SHOW GRANTS FOR 'v2b'@'localhost';
+----------------------------------------------------------------------------+
| Grants for v2b@localhost                                                   |
+----------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `v2b`@`localhost`                                    |
| GRANT ALL PRIVILEGES ON `v2board`.* TO `v2b`@`localhost` WITH GRANT OPTION |
+----------------------------------------------------------------------------+
2 rows in set (0.00 sec)

```

如果你需要導入現有的數據庫，使用以下命令：

```
mysql -p -u [用戶] [數據庫] < 備份的數據表.sql
```

如果你需要備份你現有的數據庫：

```
mysqldump -u [用戶]  -p [數據庫] > [filename].sql
```

{{< collapse summary="自動備份數據庫" >}}
來自：[Pe46dro/Bash-MySQL-Database-SFTP-FTP-Backup](https://github.com/Pe46dro/Bash-MySQL-Database-SFTP-FTP-Backup)

```bash
#!/bin/bash

# Linux MySQL Database FTP Backup Script
# Version: 1.0
# Script by: Pietro Marangon
# Skype: pe46dro
# Email: pietro.marangon@gmail.com
# SFTP function by unixfox and Pe46dro

backup_path="/root"

create_backup() {
  umask 177

  FILE="$db_name-$d.sql.gz"
  mysqldump --user=$user --password=$password --host=$host $db_name | gzip --best > $FILE

  echo 'Backup Complete'
}

clean_backup() {
  rm -f $backup_path/$FILE
  echo 'Local Backup Removed'
}

########################
# Edit Below This Line #
########################

# Database credentials

user="USERNAME HERE"
password="PASSWORD HERE"
host="IP HERE"
db_name="DATABASE NAME HERE"

# FTP Login Data
USERNAME="USERNAME HERE"
PASSWORD="PASSWORD HERE"
SERVER="IP HERE"
PORT="SERVER PORT HERE"

#Remote directory where the backup will be placed
REMOTEDIR="./"

#Transfer type
#1=FTP
#2=SFTP
TYPE=1

##############################
# Don't Edit Below This Line #
##############################

d=$(date --iso)
cd $backup_path
create_backup

if [ $TYPE -eq 1 ]
then
ftp -n -i $SERVER <<EOF
user $USERNAME $PASSWORD
binary
cd $REMOTEDIR
mput $FILE
quit
EOF
elif [ $TYPE -eq 2 ]
then
rsync --rsh="sshpass -p $PASSWORD ssh -p $PORT -o StrictHostKeyChecking=no -l $USERNAME" $backup_path/$FILE $SERVER:$REMOTEDIR
else
echo 'Please select a valid type'
fi

echo 'Remote Backup Complete'
clean_backup
#END
```

{{< /collapse >}}

# nginx

參考：[nginx: Linux packages](https://nginx.org/en/linux_packages.html#Debian)

Install the prerequisites:

```
sudo apt install curl gnupg2 ca-certificates lsb-release debian-archive-keyring
```

Import an official nginx signing key so apt could verify the packages authenticity. Fetch the key:

    curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
        | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

Verify that the downloaded file contains the proper key:

    gpg --dry-run --quiet --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg

The output should contain the full fingerprint `573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62` as follows:

    pub   rsa2048 2011-08-19 [SC] [expires: 2024-06-14]
          573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
    uid                      nginx signing key <signing-key@nginx.com>

If the fingerprint is different, remove the file.

To set up the apt repository for stable nginx packages, run the following command:

    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    http://nginx.org/packages/debian `lsb_release -cs` nginx" \
        | sudo tee /etc/apt/sources.list.d/nginx.list

Set up repository pinning to prefer our packages over distribution-provided ones:

    echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
        | sudo tee /etc/apt/preferences.d/99nginx

To install nginx, run the following commands:

    sudo apt update
    sudo apt install nginx

2. 編輯`/etc/nginx/conf.d/domain.conf`

```nginx
server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  root /var/www/domain/public;
  index index.php index.html;
  server_name domain.name;

  add_header X-Frame-Options "SAMEORIGIN";
  add_header X-XSS-Protection "1; mode=block";
  add_header X-Content-Type-Options "nosniff";
  add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";

  ssl_certificate               /etc/nginx/cert/portal/pubkey.pem;
  ssl_certificate_key           /etc/nginx/cert/portal/privkey.pem;
  ssl_protocols                 TLSv1.2 TLSv1.3;
  ssl_ciphers                   ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_session_tickets           off;
  ssl_session_cache             shared:SSL:10m;
  ssl_stapling                  on;
  ssl_stapling_verify           on;
  ssl_prefer_server_ciphers     on;

  if ($http_user_agent ~* "MicroMessenger|QBWebViewType|QQBroswer|UBrowser|360SE|360EE|MetaSr|HUAWEI|HarmonyOS|Maxthon|Sogou|LieBao|TIM|HONOR|wechatdevtools|CensysInspect|qihoobot|Baiduspider|Googlebot|Googlebot-Mobile|Googlebot-Image|Mediapartners-Google|Adsbot-Google|Feedfetcher-Google|YoudaoBot|Sosospider|MSNBot|ia_archiver"){
    return 403;
  } # 不需要屏蔽爬蟲等請移除

  location /downloads {

  }

  location / {
    try_files $uri $uri/ /index.php$is_args$query_string;
    deny 162.142.125.0/24; #如果不需要屏蔽censys.io請移除,下同
    deny 167.94.138.0/24;
    deny 167.94.145.0/24;
    deny 167.94.146.0/24;
    deny 167.248.133.0/24;
    deny 2602:80d:1000:b0cc:e::/80;
    deny 2620:96:e000:b0cc:e::/80;
  }

  location ~ .*\.(js|css)?$
  {
    expires      1h;
    error_log off;
    access_log /dev/null;
  }

  location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_index index.php;
    fastcgi_pass unix:/run/php/php7.4-fpm.sock;
  }
}

```

3.啓動及權限

在網站目錄下

```
chown -R nginx:nginx *
chmod -R 755 *
systemctl start nginx
systemctl enable nginx
```

4.  移除 Google Analysis
    編輯`resources/views`目錄下的`admin.blade.php`和`app.blade.php`
    移除或註釋其中

```php
<script async src="https://www.googletagmanager.com/gtag/js?id=G-P1E9Z5LRRK"></script>
```

# V2Board

## 最低要求

根據[官方給出的配置](https://docs.v2board.com/deploy/aapanel.html#%E4%BD%BF%E7%94%A8aapanel%E9%83%A8%E7%BD%B2)

☑️ Nginx 1.17

☑️ MySQL 5.6

☑️ PHP 7.3

1.  我的網站目錄位於`/var/www/domain`，在該目錄下：

    ```
    git clone https://github.com/v2board/v2board.git && mv v2board/* ./
    rm -rf v2board
    ```

2.  獲取 Composer

    ```
    wget https://getcomposer.org/composer-stable.phar
    mv composer-stable.phar composer.phar

    # 官方文檔使用1.90版本，如果你使用2.x版本翻車了，請使用1.90版本
    # wget https://getcomposer.org/download/1.9.0/composer.phar
    ```

    Composer v1 和 v2 的區別可以在[這裏](https://blog.packagist.com/composer-2-0-is-now-available/)詳細閱讀

3.  安裝

    ```
    php composer.phar install
    ```

        可以通過 `apt -y install php7.4-報錯缺失的名字` 解決大部分報錯。當然內存不足也會導致失敗，分配swap即可

    ```
    php artisan v2board:install
    ```

    會要求輸入數據庫信息

    數據庫用戶爲`v2b`

    用戶密碼爲`fuckGFW`

    數據庫名字爲`v2board`

    如果出現數據庫無法連結，請確認是否安裝`php7.4-mysql`

    **到此，V2Board 已經配置完成**

# V2Poseidon

{{< collapse summary="鑑於poseidon已經跑路，建議別讀了" >}}

1. 安裝

   ```
   git clone https://github.com/ColetteContreras/v2ray-poseidon.git
   cd v2ray-poseidon
   chmod +x install-release.sh
   ./install-release.sh
   ```

2. 配置

   位於`/etc/v2ray/config.json`

   ```
   {
     "poseidon": {
       "panel": "v2board",         // 这一行必须存在，且不能更改
       "nodeId": 1,                // 你的节点 ID 和 v2board 里的一致
       "checkRate": 60,            // 每隔多长时间同步一次配置文件、用户、上报服务器信息
       "webapi": "http or https://YOUR V2BOARD DOMAIN",// v2board 的域名信息
       "token": "v2board token",   // v2board 和 v2ray-poseidon 的通信密钥

       "speedLimit": 0, // 节点限速 单位 字节/s 0 表示不限速
       "user": {
         "maxOnlineIPCount": 0,  // 用户同时在线 IP 数限制 0 表示不限制
         "speedLimit": 0         // 用户限速 单位 字节/s 0 表示不限速
       },

       "localPort": 10084 // 本地 api, dokodemo-door,　监听在哪个端口，不能和服务端口相同
     }
   }
   ```

{{< /collapse >}}

## 注意

請在安裝完成 php7.4 以後註釋掉 php 的 apt repo，或者在 apt upgrade 的時候略過 php 部分，否則會造成 php 與 php-redis 等版本不同

如果出現 500 錯誤，检查站点根目录权限，递归 755，保证目录有可写文件的权限，也有可能是 Redis 扩展没有安装或者 Redis 没有按照造成的。你可以通过查看 storage/logs 下的日志来排查错误或者开启 debug 模式。

同时请确保目录的所有者为`nginx:nginx`

# References

- [How to Install MySQL on Debian 10 Linux](https://linuxize.com/post/how-to-install-mysql-on-debian-10/)
- [Install PHP 7.3 on Ubuntu 18.04 / Ubuntu 16.04 / Debian](https://computingforgeeks.com/how-to-install-php-7-3-on-ubuntu-18-04-ubuntu-16-04-debian/)
- [How to Create a Database Using MySQL from the Command Line](https://www.inmotionhosting.com/support/website/how-to-create-a-database-using-mysql-from-the-command-line/)
- [How to Import MySQL Databases in Command Line](https://www.inmotionhosting.com/support/website/how-to-import-mysql-databases-in-command-line/)
- [How to enable exec function in php.ini : Let’s figure it out](https://bobcares.com/blog/how-to-enable-exec-function-in-php-ini/)
- [How do I turn off the mysql password validation?](https://stackoverflow.com/questions/36301100/how-do-i-turn-off-the-mysql-password-validation)
- [How to use this rewrite rule in Caddy V2](https://caddy.community/t/how-to-use-this-rewrite-rule-in-caddy-v2/10460)
- [nginx error connect to php5-fpm.sock failed (13: Permission denied)](https://stackoverflow.com/questions/23443398/nginx-error-connect-to-php5-fpm-sock-failed-13-permission-denied)
- [nginx and php-fpm socket owner](https://stackoverflow.com/questions/24325695/nginx-and-php-fpm-socket-owner)
- [getting error while updating Composer](https://stackoverflow.com/questions/36979019/getting-error-while-updating-composer)
- [V2 Poseidon 官方文檔](https://poseidon-gfw.cc)
- [V2Board 官方文檔](https://docs.v2board.com)
- [Caddy 官方文檔](https://caddyserver.com/docs/caddyfile/directives/php_fastcgi)
- [How to grant all privileges to root user in MySQL 8.0](https://stackoverflow.com/a/50197630)
