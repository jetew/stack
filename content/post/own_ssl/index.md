---
title: "SSL 证书安装部署"
slug: "own_ssl"
date: 2022-12-02T10:17:55+08:00
lastmod: 2022-12-02T10:17:55+08:00
description: 在 Nginx/Caddy 服务下安装部署自有 SSL 证书
comments: false
tags:
- SSL
- Nginx
- Caddy
image: 
---

## 前言

在之前的文章中讲了在添加网站时通过 Oneinstack（LNMP 一键安装包）的 acme.sh/Caddy 来自动安装部署免费证书，当然你也可以使用自己的证书替代免费证书来进安装部署；现在来看下如何进行操作。

### 场景与前提

由于我使用的是 Oneintack 安装的 Nginx 和 Caddy 服务，所以本文以 Nginx/Caddy 视角进行操作。我的域名是在腾讯云下，并使用他们提供的为期1年免费 Trust Asia 证书，所以我以这些前提为演示；使用其他的也是完全可以的。

<!--more-->

## 操作步骤

### 下载并上传证书

首先登陆到域名控制面板，申请好你所需要的证书，然后下载到本地；如我在腾讯云下载的 Nginx 类型证书：

- 文件内容：

 ```
domain.com_nginx.zip
├ domain.com_nginx
│   ├ domain.com.scr
│   ├ domain.com.key
│   ├ domain.com_bundle.crt
│   ├ domain.com_bundle.pem
```

将其解压出来，并全部命名为 `domian.com.xxx` ；然后将 `domain.com.scr` 、`domian.com.key` 和 `domian.com.crt` 上传至 SSL 存放位置。

通过 Oneinistack 安装的位置位于 `/usr/local/nginx/conf/ssl`

### 添加网站

上传完成后，即可使用 Oneinstack 通过自有证书模式添加网站：

```bash
~/oneinstack/vhost.sh
```

输入 `2` 选择自有 SSL ，然后完成添加即可。

添加完成后可通过 `nginx -t` 进行测试，无误后即可通过 `nginx -s reload` 重载 Nginx：

```bash
nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
nginx -s reload
```

### 使用已有网站

若需要将证书部署至已有网站，则修改对应的配置文件即可，配置文件位于 `/usr/local/nginx/conf/vhost/domian.com.conf`

```bash
vim /usr/local/nginx/conf/vhost/domian.com.conf
```

按照备注位置修改证书路径：

```nginx
server {
  listen 80;
  listen [::]:80;
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  ssl_certificate /usr/local/nginx/conf/ssl/domain.com.crt;
  # 证书文件的相对路径或绝对路径
  ssl_certificate_key /usr/local/nginx/conf/ssl/domain.com.key;
  # 私钥文件的相对路径或绝对路径
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ecdh_curve X25519:prime256v1:secp384r1:secp521r1;
  ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256;
  ssl_conf_command Ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;
  ssl_conf_command Options PrioritizeChaCha;
  ssl_prefer_server_ciphers on;
  ssl_session_timeout 10m;
  ssl_session_cache shared:SSL:10m;
  ssl_buffer_size 2k;
  add_header Strict-Transport-Security max-age=15768000;
  ssl_stapling on;
  ssl_stapling_verify on;
}
```

修改完成后可通过 `nginx -t` 进行测试，无误后即可通过 `nginx -s reload` 重载 Nginx：

```bash
nginx -t
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
nginx -s reload
```

### Caddy 操作

同样将 `domian.com.key` 和 `domian.com.crt` 上传至服务器，比如我上传到 `/var/caddy/ssl` 目录下，然后对 `Caddyfile` 进行修改：

```bash
vim /etc/caddy/Caddyfile
```

按照下面进行更改：

```caddyfile
domain.com {
    log {
        output file /var/log/caddy/domain.com.log
        level error
    }
    root * /var/www/domain.com
    encode gzip
    handle_errors {
        rewrite * /{err.status_code}.html
        file_server
    }
    php_fastcgi unix//dev/shm/php-cgi.sock
    file_server
    tls /path/to/domain.com.pem /path/to/domain.com.key
    # 证书和密钥的 PEM 格式的文件绝对路径，注意中间空格
}
```

设置完毕后，通过以下命令进行测试：

```bash
caddy adapt --config /etc/caddy/Caddyfile
```

确认无误后重载配置文件：

```bash
caddy reload --config /etc/caddy/Caddyfile
```

若出现 `ERR_SSL_PROTOCOL_ERROR`，可重载 Caddy:

```bash
systemctl restart caddy
```

---

## 总结

以上就是 Nginx/Caddy 服务下安装部署 SSL 证书的详细过程，希望能对大家有所帮助。

### 参考资料

1. [Nginx 服务器 SSL 证书安装部署](https://cloud.tencent.com/document/product/400/35244)
