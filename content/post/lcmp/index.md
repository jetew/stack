---
title: "使用 Caddy+MySQL+PHP 架设 LCMP"
slug: "lcmp"
date: 2022-03-05T17:14:41+08:00
lastmod: 2022-10-27T17:14:41+08:00
description: 使用 Caddy 服务 +PHP+MySQL 架设并配置站点。
tags:
- Caddy
- Web
- MySQL
- PHP
categories:
- 学习
image: 
---

## 前言

以前一直使用的是 Hugo+Github+Netlify；即上传本地网站项目到 Github 然后通过 Netlify 进行自动部署以及发布网站。一开始还好，但是到后面就感觉比较麻烦了；每次更新文章或者后期文章有修改的话就需要重新上传部署；所以后面就改成了 LNMP+Typecho 。而最近又开始折腾起了 Caddy，虽然说 Caddy 配置简单，开箱即用，但现在一般都是 CaddyV2 ；而网上的教程与配置大多都是 CaddyV1 所以在这里进行一个记录，避免以后踩坑。

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
---

### 安装 PHP+MySQL

这里我使用的是 Oneinstack 自动安装，参考：[一键安装 LNMP 及开设站点](/p/oneinstack)

> 安装方式不限，以自己为准，我只是习惯用 Oneinstack 自动安装

`apt` 软件包安装：

```bash
apt update
apt install -y php php-fpm mysql(mariadb-server)

```

> **MySQL** 或者 **MariaDB** 都行，看个人选择进行安装，两个差别不大 (2022-10-27更新 `apt` 软件包安装)

---

### Caddy 配置

修改 Caddyfile 配置，拿不准的可以下载下来进行编辑，然后再上传，文件路径为 **/etc/caddy/Caddyfile** 

```bash
vim /etc/caddy/Caddyfile
```

我的配置如下：

```caddyfile
example.com {
    root * /var/www/xx # 网站根目录
    @key1 {
	not file
	path_regexp key1 '(.*)'
    }
    rewrite @key1 /index.php{re.key1.1} # Typecho 伪静态，建议安装完程序后写入
    encode gzip # 开启 gzip
    handle_errors {
	rewrite * /{err.status_code}.html
	file_server
    } # 错误页面
    php_fastcgi unix//dev/shm/php-cgi.sock # 使用 unix socket 通信
    file_server
    tls mail@example.com # 设置 TLS
}
```

> 这里的 unix socket 路径为 Oneinstack 自动安装方式的路径，若使用 `apt` 安装方式的话位于 `/run/phpx.x/php-fpm.sock` 
> 伪静态可以在网上查找对应程序的伪静态规则，然后通过 <a href="https://www.toolnb.com/tools/rewriteTools.html" target="_blank">伪静态转换工具</a> 进行转换，伪静态规则建议等上传并安装完程序后再写入到配置中

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
