---
title: "File Browser 在线文件管理"
slug: "filebrowser"
date: 2022-12-16T20:09:22+08:00
lastmod: 2022-12-16T20:09:22+08:00
description: 在 Linux 下使用 File Browser 搭建一个在线文件管理器
tags:
- Linux
- FileBrowser
categories:
- 学习
image: 
---

## 前言

File Browser 是一款使用 Golang 开发的文件管理器，跨平台，免费开源，功能强大。它是一个能独立运行的可执行二进制文件，可以与 Docker 或 Caddy 一起使用。这篇文章分享 Debian 11 下安装 File Browser 的过程。

## 安装配置

### 安装 File Browser

- 项目地址： [Github 项目地址](https://github.com/filebrowser/filebrowser/)

通过以下命令进行安装：

```bash
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
```

安装位置： `/usr/local/bin/filebrowser`

### 配置 File Browser

首先创建目录用于放置配置数据：

```bash
# 创建目录
mkdir /opt/filebrowser

# 创建配置数据库
filebrowser -d /opt/filebrowser/filebrowser.db config init
```

设置监听地址：

```bash
filebrowser -d /opt/filebrowser/filebrowser.db config set -a 0.0.0.0
```

设置监听端口：

```bash
filebrowser -d /opt/filebrowser/filebrowser.db config set -p 8080
```

设置根目录：

```bash
filebrowser -d /opt/filebrowser/filebrowser.db config set -r /var/filebrowser
```

设置语言环境：

```bash
filebrowser -d /opt/filebrowser/filebrowser.db config set --locale zh-cn
```

添加用户：

```bash
filebrowser -d /opt/filebrowser/filebrowser.db users add user password --perm.admin --locale zh-cn
```

其中的 `user` 与 `password` 为用户名和密码

试运行：

```bash
filebrowser -d /opt/filebrowser/filebrowser.db
```

之后即可通过 `http://ip:8080` 进行访问

---

## 后台运行及反向代理

### 后台运行

后台运行可通过 Systemd 大法：

创建 `filebrowser.service` 然后进行编辑：

```bash
vim /usr/lib/systemd/system/filebrowser.service
```

复制下面文件并粘贴：

```service
[Unit]
Description=File Browser
After=network.target

[Service]
Type=simple
ExecStart=/path/to/filebrowser -d /path/to/filebrowser.db
Restart=always

[Install]
WantedBy=multi-user.target
```

其中，`/path/to/filebrowser` 为 filebrowser 安装位置，可通过 `which filebrowser` 查询；`/path/to/filebrowser.db` 为 filebrowser 配置数据库位置。

管理命令：

```bash
# 更新配置
systemctl daemon-reload

# 启动服务
systemctl start filebrowser

# 设置开机启动
systemctl enable filebrowser
​
# 停止服务
systemctl stop filebrowser
​
# 重启服务
systemctl restart filebrowser
​
# 查看状态
systemctl status filebrowser
```

---

### 反代设置

如果不想通过 IP 地址访问，可以设置反向代理通过网址访问：

- Nginx 设置：

```nginx
location ^~ / {
  proxy_pass http://127.0.0.1:8080;
  proxy_set_header Host $http_host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-NginX-Proxy true;
  proxy_redirect off;
  client_max_body_size 10240m;
}
```

- Caddy 设置：

```caddyfile
domain.com {
    log {
        output file /var/log/caddy/filebrowser.log
        level error
    }
    root * /var/filebrowser
    header {
		Strict-Transport-Security max-age=31536000;preload
		X-Content-Type-Options nosniff
		X-Frame-Options SAMEORIGIN
	}
    encode gzip
    reverse_proxy 127.0.0.1:8080 {
        header_up Host {host}
		header_up X-Real-IP {remote}
		header_up X-Forwarded-For {remote}
		header_up X-Forwarded-Proto https
    }
    file_server
    tls user@email.com
}
```

配置完成后即可通过域名进行访问。

#### 扩展设置

如果你想要通过 `https://domian.com/pan` 进行访问，那么需要设置 `baseurl` :

```bash
filebrowser -d /opt/filebrowser/filebrowser.db config set -b /pan
```

然后进行反代：

- Ngxin：

```nginx
location ~* /pan {
  proxy_pass http://127.0.0.1:8080;
  proxy_set_header Host $http_host;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header X-NginX-Proxy true;
  proxy_redirect off;
  client_max_body_size 10240m;
}
```

- Caddy:

```caddyfile
domain.com {
    reverse_proxy /pan/* 127.0.0.1:8080 {
        header_up Host {host}
		header_up X-Real-IP {remote}
		header_up X-Forwarded-For {remote}
		header_up X-Forwarded-Proto https
    }
}
```

