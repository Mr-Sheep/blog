---
title: "V2Board + Caddy + PHP + MySQL"
date: 2020-11-12T20:56:10+08:00
draft: false
tags: ["V2Ray","V2Fly","Anti-Censorship"]
categories: ["networking","Proxy"]
---

之前寫過的[V2Board+V2poseidon](https://blog.mrsheep.xyz/posts/v2board+v2poseidon/)是基於 aaPanel(寶塔國際版) 以及一些古董配置寫的
由於寶塔面板最近[發生](https://www.php.cn/topic/bt/458726.html)重大[安全事故](https://www.chinaz.com/2020/0824/1175116.shtml)以及我大多數伺服器已經更換爲Caddy，爲了保證用戶數據安全，改用Caddy + MySQL + PHP。


<!--more-->
# Changelog
Nov 15th 2021: 添加`php7.4-bcmath`

# 開始之前
文章基於Caddy v2.2.1 + php 7.4 + mysql 5.7.32，作業系統 Debian 9
- 採用[V2Board](https://github.com/v2board/v2board/)作爲前端

- [V2-poseidon](https://github.com/ColetteContreras/v2ray-poseidon)作爲伺服器端

- 部分節點多個v2ray共存

- 其實配置很簡單


### 所需工具：

SSH + 腦子 + 電腦 + **耐心**

默認使用root用戶（不推薦）

**在點讚一下Caddy的官方社區，開發者非常熱情的幫助解決靜態的問題**

# V2Board

## 最低要求
根據[官方給出的配置](https://docs.v2board.com/deploy/aapanel.html#%E4%BD%BF%E7%94%A8aapanel%E9%83%A8%E7%BD%B2)

☑️ Nginx 1.17

☑️ MySQL 5.6

☑️ PHP 7.3 

這裏我不再使用Nginx，而目前V2board官方docker鏡像的版本還停留在Caddy V1，所以。。。。
## PHP
1. 添加Ondřej Surý的ppa:
```
wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" |  tee /etc/apt/sources.list.d/php.list
apt update
```

2. 安裝php7.4(其實只要大於7.3就好) 和 redis
```
apt -y install php7.4 php7.4-fpm php7.4-common php7.4-cli redis php7.4-bcmath
以及一大堆亂七八糟的包 -->
apt -y install php7.4-curl php7.4-redis php7.4-mysql php7.4-mbstring php7.4-json php7.4-opcache php7.4-readline php7.4-xml

```
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
   include=/etc/php/7.4/fpm/pool.d/*.conf
   include=/etc/php/7.4/fpm/www.conf
   ```

   創建並編輯`/etc/php/7.4/fpm/www.conf`

   ```
   [global]
   pid = /tmp/php-fpm.pid
   error_log = /var/log/php-fpm.log
   log_level = notice
   
   [www]
   listen = /run/php-cgi.sock
   listen.backlog = -1
   pm = dynamic
   pm.status_path = /phpfpm_status
   pm.max_children = 300
   pm.start_servers = 20
   pm.min_spare_servers = 20
   pm.max_spare_servers = 50
   request_terminate_timeout = 100
   request_slowlog_timeout = 30
   slowlog = /var/log/php-slow.log
   ```

   編輯`/etc/php/7.4/fpm/pool.d/www.conf`，添加

   ```
   user = caddy
   group = caddy
   listen.owner = caddy
   listen.group = caddy
   listen.mode = 0660
   ```

   因爲caddy使用caddy用戶運行，爲保證caddy能夠正常使用fastcgi unix socket，需要一致的用戶

   另外監聽的socket需要是非root用戶可訪問地址，如`/run/php_cgi.sock`

5. 啓動

   ```
   systemctl restart php7.4-fpm.service
   systemctl status php7.4-fpm.service
   ```

   


## MySQL
目前默認的MySQL的版本是8，在部分命令上和原有的5.X有區別

在[官方下載](https://dev.mysql.com/downloads/repo/apt/#current-tab)找到最新版本的`.deb`包
以2020年11月最新版本`mysql-apt-config_0.8.16-1_all.deb`爲例
```
wget http://repo.mysql.com/mysql-apt-config_0.8.16-1_all.deb
apt install ./mysql-apt-config_0.8.16-1_all.deb
apt update
apt install mysql-server
```
使用`systemctl status mysql`驗證mysql是否成功安裝
```
mysql.service - MySQL Community Server
   Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: en
   Active: active (running) since Wed 2020-11-11 23:42:15 EST; 11h ago
 Main PID: 23632 (mysqld)
    Tasks: 31 (limit: 4915)
   Memory: 232.3M
      CPU: 56.974s
   CGroup: /system.slice/mysql.service
           └─23632 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysql

```
### 加固 MySQL 安全
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

#### 如何移除密碼安全性驗證插件？
```
    mysql -h localhost -u root -p
    uninstall plugin validate_password;
    如果上面的命令不管用
    UNINSTALL COMPONENT 'file://component_validate_password';
```

### 創建數據庫
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
mysql> CREATE DATABASE v2board;
```


如果你需要導入現有的數據庫，使用以下命令：
```
mysql -p -u [用戶] [數據庫] < 備份的數據表.sql 
```

如果你需要備份你現有的數據庫：
```
mysqldump -u [用戶]  -p [數據庫] > [filename].sql
```

## 安裝V2Board

1. 我的網站目錄位於`/var/www/domain`，在該目錄下：

   ```
   git clone https://github.com/v2board/v2board.git && mv v2board/* ./
   rm -rf v2board
   ```

2. 獲取Composer

   

   ```
   wget https://getcomposer.org/composer-stable.phar
   mv composer-stable.phar composer.phar
   
   # 官方文檔使用1.90版本，如果你使用2.x版本翻車了，請使用1.90版本
   # wget https://getcomposer.org/download/1.9.0/composer.phar
   ```

   Composer v1和v2的區別可以在[這裏](https://blog.packagist.com/composer-2-0-is-now-available/)詳細閱讀

3. 安裝

   

   ```bash
   php composer.phar install
   ```

   會提示你不要run as root，如果提示要求確認的話輸入yes就好啦
   這個過程中如果有報錯的話，大概率是你沒有裝什麼包，基本都是`apt -y install php7.4-報錯缺失的名字`就好

   當然內存不足也會導致失敗，分配swap就好啦

   ```bash
   php artisan v2board:install
   ```

   會要求輸入數據庫信息

   數據庫用戶爲`v2b`

   用戶密碼爲`fuckGFW`

   數據庫名字爲`v2board`

   如果出現數據庫無法連結，請確認是否安裝`php7.4-mysql`

   **到此，V2Board已經配置完成**

## Caddy

1. 獲取Caddy

   以2.2.1爲例，最新版本請前往[Caddy Github Repo](https://github.com/caddyserver/caddy/releases/latest)查看

   ```
   wget https://github.com/caddyserver/caddy/releases/download/v2.2.1/caddy_2.2.1_linux_amd64.deb
   dpkg -i caddy_2.2.1_linux_amd64.deb
   ```

   



2. 編輯`/etc/caddy/Caddyfile`

   ```
   域名 {
   	tls 郵箱
   	root * /var/www/域名/public/
   	php_fastcgi  unix//run/php-cgi.sock
   	try_files {path} {path}/ /index.php?{query}
   	file_server
   }
   ```

   注意`php_fastcgi`後的`socket`需要和PHP部分配置的一樣

   如果還需要將這臺伺服器作爲節點使用，需要添加

   ```
           reverse_proxy /path/ 127.0.0.1:v2端口 {
                   header_up Host {http.request.host}
                   header_up X-Real-IP {http.request.remote.host}
                   header_up X-Forwarded-For {http.request.remote.host}
                   header_up X-Forwarded-Port {http.request.port}
                   header_up X-Forwarded-Proto {http.request.scheme}
           }
   ```

   完整的`Caddyfile`

   ```
   域名 {
   	tls 郵箱
   	root * /var/www/域名/public/
   	php_fastcgi  unix//run/php-cgi.sock
   	try_files {path} {path}/ /index.php?{query}
   	file_server
   	reverse_proxy /path/ 127.0.0.1:v2端口 {
     	header_up Host {http.request.host}
       header_up X-Real-IP {http.request.remote.host}
       header_up X-Forwarded-For {http.request.remote.host}
       header_up X-Forwarded-Port {http.request.port}
       header_up X-Forwarded-Proto {http.request.scheme}
     }
   }
   ```

   3.啓動及權限

   在網站目錄下

   ```
   chown -R caddy:caddy *
   chmod -R 755 *
   systemctl start caddy
   systemctl enable caddy
   ```

   4. 移除Google Analysis
   編輯`resources/views`目錄下的`admin.blade.php`和`app.blade.php`
   移除或註釋其中
   ```
   <script async src="https://www.googletagmanager.com/gtag/js?id=G-P1E9Z5LRRK"></script>
   ```

# V2Poseidon

1. 安裝

   我討厭docker，如果喜歡docker的話請移步[官綱文檔](https://poseidon-gfw.cc/shi-yong-v2ray-poseidon/v2ray-poseidon-with-v2board)

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

3. 在V2Board中添加新的節點

   建議參考[之前的文章](https://blog.mrsheep.xyz/posts/v2board+v2poseidon/)和[官綱文檔](https://poseidon-gfw.cc/shi-yong-v2ray-poseidon/v2ray-poseidon-with-v2board)

## 注意

請在安裝完成php7.4以後註釋掉php的apt repo，或者在apt upgrade的時候略過php部分，否則會造成php與php-redis等版本不同

如果出現500錯誤，检查站点根目录权限，递归755，保证目录有可写文件的权限，也有可能是Redis扩展没有安装或者Redis没有按照造成的。你可以通过查看storage/logs下的日志来排查错误或者开启debug模式。

同时请确保目录的所有者为`caddy:caddy`


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
- [V2 Poseidon官方文檔](https://poseidon-gfw.cc)
- [V2Board 官方文檔](https://docs.v2board.com)
- [Caddy 官方文檔](https://caddyserver.com/docs/caddyfile/directives/php_fastcgi)
- [How to grant all privileges to root user in MySQL 8.0](https://stackoverflow.com/a/50197630)