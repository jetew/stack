---
title: "使用 Caddy + MariaDB + PHP 架设 LCMP"
slug: "lcmp"
date: 2022-03-05T17:14:41+08:00
lastmod: 2022-10-27T17:14:41+08:00
description: 使用 Caddy + PHP + MariaDB 架设并配置站点。
comments: false
tags:
- Linux
- Caddy
- Web
- MariaDB
- PHP
image: 
---

## 前言

以前一直使用的是 Hugo + Github + Vercel；即上传本地网站项目到 Github 然后通过 Vercel 进行自动部署以及发布网站。一开始还好，但是到后面就感觉比较麻烦了；每次更新文章或者后期文章有修改的话就需要重新上传部署；所以后面就改成了 Typecho。而最近又开始折腾起了 Caddy，需要搭建 LCMP 环境，所以在这里进行一个记录，避免以后踩坑。

<!--more-->

---

## 安装及配置
### 安装Caddy

我使用的是 Debian 10 系统

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install caddy
```

### 安装 MariaDB

可以使用 Oneinstack 自动安装，参考：[一键安装 LNMP 及开设站点](/archives/oneinstack)

这里主要讲讲通过 `apt` 包管理器安装

首先安装 MariaDB：

```bash
apt update
apt install -y mariadb-server mariadb-common mariadb-client

```

> **MySQL** 或者 **MariaDB** 都行，看个人选择进行安装，两个差别不大

按照下面简单修改下配置，文件位于 `/etc/mysql/mariadb.conf.d/50-server.cnf`

```bash
[client]
default-character-set = utf8mb4

[mysqld]
innodb_buffer_pool_size = 64M
max_allowed_packet = 500M
net_read_timeout = 3600
net_write_timeout = 3600

[mariadb]
character-set-server = utf8mb4

[client-mariadb]
default-character-set = utf8mb4
```

启动 MariaDB 并进行安全设置

```bash
systemctl start mariadb
mysql_secure_installation
```

```bash
# 输入初始密码，回车即可
Enter current password for root (enter for none): 
OK, successfully used password, moving on...

# 设置密码
Set root password? [Y/n] y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!

# 移除匿名用户
Remove anonymous users? [Y/n] y
 ... Success!

# 是否禁止非本地登录
Disallow root login remotely? [Y/n] n
 ... skipping.

# 是否移除测试数据
Remove test database and access to it? [Y/n] y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

# 是否重新刷新权限
Reload privilege tables now? [Y/n] y
 ... Success!

Cleaning up...

Thanks for using MariaDB!
```

### 安装 PHP

通过 `apt list php*` 可以看到 Debian10 下 PHP 版本为 7.3，然后来安装 PHP 及其组件

```bash
apt update
apt install -y php7.3 php7.3-bcmath php7.3-bz2 php7.3-cgi php7.3-cli php7.3-common php7.3-curl php7.3-dba php7.3-enchant php7.3-fpm php7.3-gd php7.3-gmp php7.3-imap php7.3-interbase php7.3-intl php7.3-json php7.3-ldap php7.3-mbstring php7.3-mysql php7.3-odbc php7.3-opcache php7.3-pgsql php7.3-phpdbg php7.3-pspell php7.3-readline php7.3-recode php7.3-snmp php7.3-soap php7.3-sqlite3 php7.3-sybase php7.3-tidy php7.3-xml php7.3-xmlrpc php7.3-xsl php7.3-zip
```

按照下面简单修改下配置文件，位于 `/etc/php/7.3/fpm/pool.d/www.conf`

```bash
user = caddy
group = caddy
listen.owner = caddy
listen.group = caddy
listen.acl_users = caddy
listen.allowed_clients = 127.0.0.1
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
slowlog = /var/log/www-slow.log
php_admin_value[error_log] = /var/log/www-error.log
php_admin_flag[log_errors] = on

# 最后添加如下
php_value[session.save_handler] = files
php_value[session.save_path]    = /var/lib/php/session
php_value[soap.wsdl_cache_dir]  = /var/lib/php/wsdlcache
php_value[opcache.file_cache]   = /var/lib/php/opcache
```

然后按照下面简单修改下 `php.ini`，位于 `/etc/php/7.3/fpm/php.ini`

```bash
disable_functions = passthru,exec,shell_exec,system,chroot,chgrp,chown,proc_open,proc_get_status,ini_alter,ini_alter,ini_restore
max_execution_time = 300
max_input_time = 300
post_max_size = 128M
upload_max_filesize = 128M
expose_php = Off
short_open_tag = On
mysqli.default_socket = /run/mysqld/mysqld.sock
pdo_mysql.default_socket = /run/mysqld/mysqld.sock
```

设置下权限

```bash

```
chown caddy:caddy /var/lib/php/session
chown caddy:caddy /var/lib/php/wsdlcache
chown caddy:caddy /var/lib/php/opcache
---

### Caddy 配置

修改 Caddyfile 配置，拿不准的可以下载下来进行编辑，然后再上传，文件路径为 **/etc/caddy/Caddyfile** 

```bash
vim /etc/caddy/Caddyfile
```

我的配置如下：

```caddy
example.com {

	# 网站根目录
	root * /var/www/xx

	# Typecho 伪静态，建议安装完程序后写入
	@key1 {
		not file
		path_regexp key1 '(.*)'
	}
	rewrite @key1 /index.php{re.key1.1}

	# 开启 gzip
	encode gzip

	# 错误页面
	handle_errors {
		rewrite * /{err.status_code}.html
		file_server
	}

	# 使用 unix socket 通信
	php_fastcgi /run/php/php7.3-fpm.sock
	file_server

	# 设置 TLS
	tls mail@example.com
}
```

> 若使用 Oneinstack 自动安装方式，php_fastcgi 设置为 `unix//dev/shm/php-cgi.sock` 
> 伪静态可以在网上查找对应程序的伪静态规则，然后通过 [伪静态转换工具](https://www.toolnb.com/tools/rewriteTools.html) 进行转换，伪静态规则建议等上传并安装完程序后再写入到配置中

配置完成后使用 `caddy adapt` 命令进行检查：

```bash
caddy adapt --config /etc/caddy/Caddyfile
```

确认无报错后，重载 Caddyfile ：

```bash
caddy reload --config /etc/caddy/Caddyfile
```

重启 Caddy 并添加自启动

```bash
systemctl restart caddy
systemctl enable caddy
```

---

## 写在最后

- Caddy 作为 Golang 编写的程序，简单易用，但是性能方面是稍落后于 Nginx 的，但是对于小站来说，这一点点的差距是可以忽略不计的。

- PHP 和 MySQL 可以根据自己的喜好来安装，或者选择 Mariadb、SQLite 等等都是可以的。
