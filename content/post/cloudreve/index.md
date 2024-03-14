---
title: "使用 Cloudreve 搭建文件管理"
slug: "cloudreve"
date: 2023-01-02T13:21:30+08:00
lastmod: 2023-01-02T13:21:30+08:00
description: Linux 下使用 Cloudreve 搭建在线文件管理
comments: false
tags:
- Linux
- FileBrowser
image: 
---

## 前言

Cloudreve 可以让您快速搭建起公私兼备的网盘系统。Cloudreve 在底层支持不同的云存储平台，用户在实际使用时无须关心物理存储方式。你可以使用 Cloudreve 搭建个人用网盘、文件分享系统，亦或是针对大小团体的公有云系统。

## 安装配置

### 获取 Cloudreve

- 项目地址：[Github 项目地址](https://github.com/cloudreve/Cloudreve)

你可以在 [GitHub Release](https://github.com/cloudreve/Cloudreve/releases) 页面获取已经构建打包完成的主程序。其中每个版本都提供了常见系统架构下可用的主程序，命名规则为cloudreve_版本号_操作系统_CPU架构.tar.gz 。比如，普通 64 位 Linux 系统上部署 3.0.0 版本，则应该下载cloudreve_3.0.0_linux_amd64.tar.gz。

### 安装启动

通过上一步获取 Cloudreve 下载地址后，即可下载至本地并启动(我这里最新版本是3.6.2)：

```bash
#创建文件夹
mkdir -p /opt/cloudreve

#下载程序
wget https://github.com/cloudreve/Cloudreve/releases/download/3.6.2/cloudreve_3.6.2_linux_amd64.tar.gz

#解压程序
tar zxvf cloudreve_3.6.2_linux_amd64.tar.gz

#赋予执行权限
chmod +x ./cloudreve

#启动 Cloudreve
./cloudreve
```

Cloudreve 在首次启动时，会创建初始管理员账号，请注意保管管理员密码，此密码只会在首次启动时出现。如果您忘记初始管理员密码，需要删除同级目录下的 cloudreve.db，重新启动主程序以初始化新的管理员账户。

Cloudreve 默认会监听 5212 端口。你可以在浏览器中访问 http://IP:5212 进入 Cloudreve。

以上步骤操作完后，最简单的部署就完成了。你可能需要一些更为具体的配置，才能让 Cloudreve 更好的工作，具体流程请参考下面的配置流程。

### 反向代理

在自用或者小规模使用的场景下，你完全可以使用 Cloudreve 内置的 Web 服务器。但是如果你需要使用 HTTPS，亦或是需要与服务器上其他 Web 服务共存时，你可能需要使用主流 Web 服务器反向代理 Cloudreve ，以获得更丰富的扩展功能。

你需要在 Web 服务器中新建一个虚拟主机，完成所需的各项配置（如启用 HTTPS），然后在网站配置文件中加入反代规则：

#### Nginx 配置

在网站的 server 字段中加入：

```nginx
location / {
	proxy_pass http://127.0.0.1:5212;
	proxy_set_header Host $http_host;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-Proto $scheme;
	proxy_set_header X-NginX-Proxy true;
	proxy_redirect off;
	# 如果您要使用本地存储策略，请将下一行注释符删除，并更改大小为理论最大文件尺寸
	# client_max_body_size 20000m;
}
```

#### Apache 配置

在 VirtualHost 字段下加入反代配置项 ProxyPass，比如：

```apache
<VirtualHost *:80>
	ServerName myapp.example.com
	ServerAdmin webmaster@example.com
	DocumentRoot /www/myapp/public
	
	# 以下为关键部分
	AllowEncodedSlashes NoDecode
	ProxyPass "/" "http://127.0.0.1:5212/" nocanon
</VirtualHost>
```
#### Caddy 配置

在网站配置中加入：

```caddyfile
domain.com {
		reverse_proxy 127.0.0.1:5212 {
				header_up Host {host}
				header_up X-Real-IP {remote}
				header_up X-Forwarded-For {remote}
				header_up X-Forwarded-Proto https
		}
}
```

### 后台运行

这里依旧使用 Systemd 大法：

创建编辑配置文件：

```bash
vim /usr/lib/systemd/system/cloudreve.service
```

将下面的 `PATH_TO_CLOUDREVE` 换成程序所在目录：

```service
[Unit]
Description=Cloudreve
After=network.target
After=mysqld.service
Wants=network.target

[Service]
WorkingDirectory=/PATH_TO_CLOUDREVE
ExecStart=/PATH_TO_CLOUDREVE/cloudreve
Restart=on-abnormal
RestartSec=5s
KillMode=mixed

StandardOutput=null
StandardError=syslog

[Install]
WantedBy=multi-user.target
```

管理命令：

```bash
# 更新配置
systemctl daemon-reload

# 启动服务
systemctl start cloudreve

# 设置开机启动
systemctl enable cloudreve

# 停止服务
systemctl stop cloudreve

# 重启服务
systemctl restart cloudreve

# 查看状态
systemctl status cloudreve
```

## 扩展配置

默认情况下，Cloudreve 会使用内置的 SQLite 数据库，并在同级目录创建数据库文件cloudreve.db，如果您想要使用 MySQL，可以按照以下步骤配置，并重启 Cloudreve。注意，Cloudreve 只支持大于或等于 5.7 版本的 MySQL 。

### 创建配置文件

```bash
cd /opt/cloudreve
vim conf.ini
```

加入如下配置，并进行修改：

```ini
[System]
# 运行模式
Mode = master
# 监听端口
Listen = :5212
# 是否开启 Debug
Debug = false
# Session 密钥, 一般在首次启动时自动生成
SessionSecret = 23333
# Hash 加盐, 一般在首次启动时自动生成
HashIDSalt = something really hard to guss

# 数据库相关
[Database]
# 数据库类型，目前支持 sqlite/mysql/mssql/postgres
Type = mysql
# MySQL 端口
Port = 3306
# 用户名
User = root
# 密码
Password = password
# 数据库地址
Host = 127.0.0.1
# 数据库名称
Name = root
# 数据表前缀
TablePrefix = cd_
# 字符集
Charset = utf8mb4
# 进程退出前安全关闭数据库连接的缓冲时间
GracePeriod = 30
```

设置完成后将 Systemd 文件修改如下：

```systemd
[Unit]
Description=Cloudreve
After=network.target
After=mysqld.service
Wants=network.target

[Service]
WorkingDirectory=/PATH_TO_CLOUDREVE
ExecStart=/PATH_TO_CLOUDREVE/cloudreve -c /path/to/conf.ini
Restart=on-abnormal
RestartSec=5s
KillMode=mixed

StandardOutput=null
StandardError=syslog

[Install]
WantedBy=multi-user.target
```

其中 `/path/to/conf.ini` 为配置文件所在位置，我这里是 `/opt/cloudreve/conf.ini`

然后更新配置并启动服务：

```bash
systemctl daemon-reload
systemctl start cloudreve
```
