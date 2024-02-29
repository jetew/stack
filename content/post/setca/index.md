---
title: "指定签发 SSL CA 机构"
slug: "setca"
date: 2022-11-24T09:32:18+08:00
lastmod: 2022-11-24T09:32:18+08:00
description: LNMP/Caddy 添加网站时指定签发 SSL  CA 机构
tags:
- SSL
- Nginx
- Caddy
image: 
---

## 前言

前几天使用 Oneinstack 添加网站进行临时测试，在自动签发 SSL 时发现证书签发失败；找了很久尝试了很多解决方法，但是并没有作用。。。然后想有没有可能是因为在同一 IP 多次申请 SSL 证书然后被限制了；但是我并没有进行多次申请。。。不管怎么样还是更换一下 CA 机构看看能不能解决，没想竟然成功了；虽然没有找到问题的原因，但最终还是解决了。遂在此进行一个记录。

<!--more-->

## 指定 CA 机构

### 服务器环境

我的服务器使用的是 Debian 10/11 ，Web 服务使用的是 Nginx/Caddy ；LNMP 环境由 Oneinstack 自动安装

### Nginx

由于 Oneinstack 集成了 acme.sh ，所以直接更换 acme.sh 的默认 CA 即可（我之前的是 Let's Encrypt ，现在换成 ZeroSSL ）：

```bash
cd ~/.acme.sh
acme.sh --set-default-ca --server zerossl
```

CA 机构：

|    CA 机构    |有效期| ECC |   域名数  |通配符|IPV4|IPV6|
|     :--:      | :--:|:--: |    :--:   | :--:|:--:|:--:|
| Let's Encrypt |  90 | Yes |    100    | Yes | No | No |
|    ZeroSSL    |  90 | Yes |    100    | Yes | No | No |
|     Google    |  90 | Yes |    100    | Yes | No | No |
|    Buypass    | 180 | Yes |     5     | Paid| No | No |
|    SSL.com    |  90 | Yes |      2    | Paid| No | No |
|      HiCA     | 180 | Paid|10 (1通配符)|  Yes| Yes| Yes|

LNMP 一键安装包的话，可以使用上述方式指定默认 CA ，也可以在添加网站时手动选择 CA

### Caddy

Caddy 比较简单，由于它是自动开启 HTTPS 的。在编辑 **Caddyfile** 添加网站时，只需要填上域名，它就会自动签发证书：

```caddyfile
domain.com {

}
```

或者你也可以设置 `tls` 邮箱，Caddy 会在签发证书遇到问题时自动更换 CA：

```caddyfile
domain.com {
    tls your@email.com
}
```

如果你需要指定 CA 机构为 ZeroSSL，你可以进行全局设置：

```caddyfile
{
    acme_ca https://acme.zerossl.com/v2/DV90
    email your@email.com
}

domain.com {

}
```

或者单独对某个网站进行设置：

```caddyfile
domain.com {
    tls your@email.com {
        ca https://acme.zerossl.com/v2/DV90
    }
}
```

如果要指定 CA 为 Let’s Encrypt，只需要将 `https://acme.zerossl.com/v2/DV90` 更换为 `https://acme-v02.api.letsencrypt.org/directory` 即可