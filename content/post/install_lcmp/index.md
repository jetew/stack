---
title: "Debian 下手动安装 LCMP 环境"
slug: "install_lcmp"
description: "放弃 LNMP 一键包和 Oneinstack 手动安装及配置 LCMP 环境"
date: 2023-10-20T22:24:35+08:00
lastmod: 2023-10-20T22:24:35+08:00
draft: false
toc: true
comments: false
image: ""
tags: ["Linux","Caddy","MariaDB","PHP","Web"]
---

## 前言

继 [LNMP 供应链投毒事件](https://mp.weixin.qq.com/s/OT7C1l5rjBNCawFXRIUJOQ) 后，知名 LNMP 部署工具 Oneinstack 也被发现挂码，参考 Oneinstack 项目 [Issues #487](https://github.com/oneinstack/oneinstack/issues/487) 和 [Issues #511](https://github.com/oneinstack/oneinstack/issues/511)

此事在 Github 和 V2EX 上已有帖子在讨论；而始作俑者 **金华市矜贵网络科技有限公司** 至今陆续拥有了 **WDCP**、**LNMP 一建包** 和 **Oneinstack**；鉴于金华矜贵还试图收购秋水逸冰大佬的 **LAMP 一键安装包** (但被秋大拒绝了)，他们后续可能还会盯上其他部署工具；所以建议大家最近有使用过上述工具的直接重装系统。

而对 LNMP 一建包和 Oneinstack 有使用需求的人不在少数，所以我们必须寻找替代品，如上文提到的秋水逸冰大佬的 LAMP/LCMP 一键包或者学着自己进行搭建与配置。

---

### 脚本推荐

#### LAMP 一键包

LAMP 一键安装包是一个用 Linux Shell 编写的可以为 Amazon Linux 2/CentOS/Debian/Ubuntu 系统的 VPS 或服务器安装 LAMP(Linux + Apache + MySQL/MariaDB + PHP) 生产环境的 Shell 脚本。

  - [LAMP 一键包官网](https://lamp.sh)
  - [Github 项目地址](https://github.com/teddysun/lamp)

#### LCMP 一键包

LCMP 一键包 (Linux + Caddy2 + MySQL/MariaDB + PHP) 是一个强大的 Bash 脚本，用于安装 Caddy2 + MariaDB + PHP；可以通过 `yum` 或 `apt-get` 命令在内存较小的 VPS 中安装 Caddy2 + MariaDB + PHP，只需在安装前输入数字选择要安装的内容即可；同为秋水逸冰大佬制作

  - [Github 项目地址](https://github.com/teddysun/lamp)

接下来看下如何手动安装及配置。

---

## 安装与配置
### 安装必要软件和依赖

```bash
apt update
apt install -y debian-keyring debian-archive-keyring build-essential gcc g++ make cmake autoconf libjpeg62-turbo-dev libjpeg-dev libpng-dev libwebp7 libwebp-dev libfreetype6 libfreetype6-dev libssh2-1-dev libmhash2 libpcre3 libpcre3-dev gzip libbz2-1.0 libbz2-dev libgd-dev libxml2 libxml2-dev libsodium-dev argon2 libargon2-1 libargon2-dev libiconv-hook-dev zlib1g zlib1g-dev libc6 libc6-dev libc-client2007e-dev libglib2.0-0 libglib2.0-dev bzip2 libzip-dev libbz2-1.0 libncurses5 libncurses5-dev libaio1 libaio-dev numactl libreadline-dev curl libcurl3-gnutls libcurl4-openssl-dev e2fsprogs libkrb5-3 libkrb5-dev libltdl-dev libidn11-dev openssl net-tools libssl-dev libtool libevent-dev bison re2c libsasl2-dev libxslt1-dev libicu-dev locales patch vim zip unzip tmux htop bc dc expect libexpat1-dev libonig-dev libtirpc-dev rsync git lsof lrzsz rsyslog cron logrotate chrony libsqlite3-dev psmisc wget sysv-rc apt-transport-https ca-certificates software-properties-common gnupg screen
```

### 安装 Caddy

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install caddy
```

#### 配置 Caddy

首先为 Caddy 创建网站目录和 SSL 存放目录，网站目录我设置为 **/data/www**，自有证书存放目录在 Caddy 默认位置中创建一个文件夹，位置可以自己更改

```bash
mkdir -p /data/www/default
mkdir -p /var/lib/caddy/ssl
```

将 Caddy 默认网站文件移到创建的默认网站文件夹中

```bash
mv /usr/share/caddy/index.html /data/www/default
rm -r /usr/share/caddy
```

编辑 Caddyfile 配置文件，并进行重写，位于 **/etc/caddy/Caddyfile**

```bash
cat > /etc/caddy/Caddyfile << EOF
:80 {
	root * /data/www/default
	header {
		Strict-Transport-Security max-age=31536000;preload
		X-Content-Type-Options nosniff
		X-Frame-Options SAMEORIGIN
	}
	encode gzip
	php_fastcgi unix//dev/shm/php-cgi.sock
	file_server
}
EOF
caddy fmt --overwrite /etc/caddy/Caddyfile
```

---

### 安装 MariaDB

可以使用预编译的二进制包安装或者源码编译进行安装，自行选择即可；我推荐使用二进制包进行安装。

#### 安装前的准备

为 MariaDB 创建用户组，创建安装目录并设置权限

```bash
useradd -M -s /sbin/nologin mysql
mkdir -p /usr/local/mariadb
mkdir -p /data/mariadb
chown -R mysql:mysql /usr/local/mariadb
chown -R mysql:mysql /data/mariadb
```

编译安装 Jemalloc 为 MariaDB 提供内存分配管理

```bash
cd /usr/local/src
wget https://github.com/jemalloc/jemalloc/archive/5.3.0.tar.gz
tar zxf 5.3.0.tar.gz
cd jemalloc-5.3.0
autoconf
./configure
make -j$(nproc)
make install
```

链接动态库
```bash
ln -s /usr/local/lib/libjemalloc.so.2 /usr/lib/libjemalloc.so.1
echo '/usr/local/lib' > /etc/ld.so.conf.d/jemalloc.conf
ldconfig
```

#### 二进制安装

下载二进制包

```bash
cd /usr/local/src
wget https://archive.mariadb.org/mariadb-10.11.7/bintar-linux-systemd-x86_64/mariadb-10.11.7-linux-systemd-x86_64.tar.gz
```

国内主机可以使用清华源地址：

```
https://mirrors.tuna.tsinghua.edu.cn/mariadb/mariadb-10.11.7/bintar-linux-systemd-x86_64/mariadb-10.11.7-linux-systemd-x86_64.tar.gz
```

只需要将下载好的预编译二进制文件解压到安装位置，然后更改相关文件的配置即可

```bash
cd /usr/local/src
tar zxf mariadb-10.11.7-linux-systemd-x86_64.tar.gz
mv /usr/local/src/mariadb-10.11.7-linux-systemd-x86_64/* /usr/local/mariadb

# 为 MariaDB 开启 Jemalloc 支持
sed -i 's@executing mysqld_safe@executing mysqld_safe\nexport LD_PRELOAD=/usr/local/lib/libjemalloc.so@' /usr/local/mariadb/bin/mysqld_safe

# 更改相关文件中 MariaDB 安装位置
sed -i "s@/usr/local/mysql@/usr/local/mariadb@g" /usr/local/mariadb/bin/mysqld_safe
```

#### 配置 MariaDB

首先设置 Service 脚本，方便进行管理；这里主要设置一下安装路径和数据路径

```bash
cp /usr/local/mariadb/support-files/mysql.server /etc/init.d/mysql

# 更改 MariaDB安装位置
sed -i "s@^basedir=.*@basedir=/usr/local/mariadb@" /etc/init.d/mysql

# 更改 MariaDB 数据位置
sed -i "s@^datadir=.*@datadir=/data/mariadb@" /etc/init.d/mysql
chmod +x /etc/init.d/mysql
```

接着来配置下 `my.cnf` 文件

```bash
cat > /etc/my.cnf << EOF
[client]
port = 3306
socket = /tmp/mysql.sock
default-character-set = utf8mb4

[mysqld]
port = 3306
socket = /tmp/mysql.sock

basedir = /usr/local/mariadb
datadir = /data/mariadb
pid-file = /data/mariadb/mysql.pid
user = mysql
bind-address = 0.0.0.0
server-id = 1

init-connect = 'SET NAMES utf8mb4'
character-set-server = utf8mb4

skip-name-resolve
#skip-networking
back_log = 300

max_connections = 1000
max_connect_errors = 6000
open_files_limit = 65535
table_open_cache = 128
max_allowed_packet = 500M
binlog_cache_size = 1M
max_heap_table_size = 8M
tmp_table_size = 16M

read_buffer_size = 2M
read_rnd_buffer_size = 8M
sort_buffer_size = 8M
join_buffer_size = 8M
key_buffer_size = 4M

thread_cache_size = 8

query_cache_type = 1
query_cache_size = 8M
query_cache_limit = 2M

ft_min_word_len = 4

log_bin = mysql-bin
binlog_format = mixed
expire_logs_days = 7

log_error = /data/mariadb/mysql-error.log
slow_query_log = 1
long_query_time = 1
slow_query_log_file = /data/mariadb/mysql-slow.log

performance_schema = 0

#lower_case_table_names = 1

skip-external-locking

default_storage_engine = InnoDB
innodb_file_per_table = 1
innodb_open_files = 500
innodb_buffer_pool_size = 64M
innodb_write_io_threads = 4
innodb_read_io_threads = 4
innodb_purge_threads = 1
innodb_flush_log_at_trx_commit = 2
innodb_log_buffer_size = 2M
innodb_log_file_size = 32M
innodb_max_dirty_pages_pct = 90
innodb_lock_wait_timeout = 120

bulk_insert_buffer_size = 8M
myisam_sort_buffer_size = 8M
myisam_max_sort_file_size = 10G

interactive_timeout = 28800
wait_timeout = 28800

[mysqldump]
quick
max_allowed_packet = 500M

[myisamchk]
key_buffer_size = 8M
sort_buffer_size = 8M
read_buffer = 4M
write_buffer = 4M
EOF
```

然后进行如下修改：

```bash
# 最大连接数，改为内存（单位：M）/3
max_connections = 

# 内存 1500M - 2500M 改成如下：
thread_cache_size = 16
query_cache_size = 16M
myisam_sort_buffer_size = 16M
key_buffer_size = 16M
innodb_buffer_pool_size = 128M
tmp_table_size = 32M
table_open_cache = 256

# 内存 2500M - 3500M 改成如下：
thread_cache_size = 32
query_cache_size = 32M
myisam_sort_buffer_size = 32M
key_buffer_size = 64M
innodb_buffer_pool_size = 512M
tmp_table_size = 64M
table_open_cache = 512

# 内存大于 3500M 改成如下：
thread_cache_size = 64
query_cache_size = 64M
myisam_sort_buffer_size = 64M
key_buffer_size = 256M
innodb_buffer_pool_size = 1024M
tmp_table_size = 128M
table_open_cache = 1024
```

创建必要文件、文件夹，并赋予权限

```bash
touch /tmp/mysql.sock
chown mysql:mysql /tmp/mysql.sock
mkdir -p /data/mariadb
touch /data/mariadb/{mysql.pid,mysql-error.log,mysql-slow.log}
chown -R mysql:mysql /data/mariadb
chmod 664 /tmp/mysql.sock /data/mariadb/*
chown -R mysql:mysql /usr/local/mariadb
```

添加环境变量

```bash
echo "PATH=/usr/local/mariadb/bin:\$PATH" > /etc/profile.d/mysql.sh
source /etc/profile.d/mysql.sh
```

接下来使用自带的脚本进行数据导入及初始化

```bash
/usr/local/mariadb/scripts/mysql_install_db --user=mysql --basedir=/usr/local/mariadb/ --datadir=/data/mariadb
```

![初始化数据](1.png)

完成之后，赋予 my.cnf 对应的权限并启动 MariaDB

```bash
chmod 600 /etc/my.cnf
systemctl daemon-reload
service mysql start
```

然后执行下面命令进行安全设置，`dbrootpwd=` 设置为你要设置的密码

```bash
dbrootpwd="password"
/usr/local/mariadb/bin/mysql -e "grant all privileges on *.* to root@'127.0.0.1' identified by \"${dbrootpwd}\" with grant option;"
/usr/local/mariadb/bin/mysql -e "grant all privileges on *.* to root@'localhost' identified by \"${dbrootpwd}\" with grant option;"
/usr/local/mariadb/bin/mysql -uroot -p${dbrootpwd} -e "delete from mysql.user where Password='' and User not like 'mariadb.%';"
/usr/local/mariadb/bin/mysql -uroot -p${dbrootpwd} -e "delete from mysql.db where User='';"
/usr/local/mariadb/bin/mysql -uroot -p${dbrootpwd} -e "delete from mysql.proxies_priv where Host!='localhost';"
/usr/local/mariadb/bin/mysql -uroot -p${dbrootpwd} -e "drop database test;"
/usr/local/mariadb/bin/mysql -uroot -p${dbrootpwd} -e "reset master;"
```

完成安全设置后为 MariaDB 链接动态库

```bash
echo "/usr/local/mariadb/lib" > /etc/ld.so.conf.d/mariadb.conf
ldconfig
```

---

### 安装 PHP
#### 准备工作

在安装 PHP 之前，需要编译安装 libiconv 

```bash
cd /usr/local/src
wget https://ftp.gnu.org/gnu/libiconv/libiconv-1.17.tar.gz
tar zxf libiconv-1.17.tar.gz
cd libiconv-1.17
./configure
make -j$(nproc)
make install
```

链接动态库

```bash
echo '/usr/local/lib' > /etc/ld.so.conf.d/libc.conf
ldconfig
```

#### 源码编译安装

```bash
cd /usr/local/src
wget https://secure.php.net/distributions/php-8.2.16.tar.gz
tar zxf php-8.2.16.tar.gz
cd php-8.2.16
mkdir build-php && cd build-php
../configure --prefix=/usr/local/php \
--with-config-file-path=/usr/local/php/etc \
--with-config-file-scan-dir=/usr/local/php/etc/php.d \
--with-fpm-user=caddy \
--with-fpm-group=caddy \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--with-iconv=/usr/local/ \
--with-freetype \
--with-jpeg \
--with-zlib \
--with-password-argon2 \
--with-sodium \
--with-curl \
--with-openssl \
--with-mhash \
--with-xsl \
--with-gettext \
--with-zip \
--enable-fpm \
--enable-opcache \
--enable-mysqlnd \
--enable-xml \
--enable-bcmath \
--enable-shmop \
--enable-exif \
--enable-sysvsem \
--enable-mbregex \
--enable-mbstring \
--enable-gd \
--enable-pcntl \
--enable-sockets \
--enable-ftp \
--enable-intl \
--enable-soap \
--disable-fileinfo \
--disable-rpath \
--disable-debug
make ZEND_EXTRA_LIBS='-liconv' -j $(nproc)
make install
```

为 PHP 添加环境变量

```bash
echo "export PATH=/usr/local/php/bin:\$PATH" > /etc/profile.d/php.sh
source /etc/profile.d/php.sh
```

复制 php.ini 配置并进行修改

```bash
cp /usr/local/src/php-8.2.16/php.ini-production /usr/local/php/etc/php.ini

# 修改以下部分
memory_limit = 
output_buffering = On
short_open_tag = On
expose_php = Off
equest_order = "CGP"
date.timezone = Asia/Shanghai
post_max_size = 100M
upload_max_filesize = 50M
max_execution_time = 600
realpath_cache_size = 2M
disable_functions = passthru,exec,system,chroot,chgrp,chown,shell_exec,proc_open,proc_get_status,ini_alter,ini_restore,dl,readlink,symlink,popepassthru,stream_socket_server,fsocket,popen
```

其中 `memory_limit` 项参数如下：

```bash
# 内存小于 640M
memory_limit = 64

# 内存为 640M - 1280M
memory_limit = 128

# 内存为 1280M - 2500M
memory_limit = 192

# 内存为 2500M - 3500M
memory_limit = 256

# 内存为 3500M - 4500M
memory_limit = 320

# 内存为 4500M - 8000M
memory_limit = 384

#内存大于 8000M
memory_limit = 448
```

接下来为 PHP 开启 OPcache 缓存并进行配置

```bash
cat > /usr/local/php/etc/php.d/opcache.ini << EOF
[opcache]
zend_extension=opcache.so
opcache.enable=1
opcache.enable_cli=1
opcache.memory_consumption=
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=100000
opcache.max_wasted_percentage=5
opcache.use_cwd=1
opcache.validate_timestamps=1
opcache.revalidate_freq=60
;opcache.save_comments=0
opcache.consistency_checks=0
;opcache.optimization_level=0
EOF
```

其中 `opcache.memory_consumption=` 项，与上文 **php.ini** 中 `memory_limit` 一致



添加 php-fpm 启动脚本，并设置开机自启

```bash
cat > /usr/lib/systemd/system/php-fpm.service << 'EOF'
[Unit]
Description=The PHP FastCGI Process Manager
Documentation=http://php.net/docs.php
After=network.target

[Service]
Type=simple
PIDFile=/usr/local/php/var/run/php-fpm.pid
ExecStart=/usr/local/php/sbin/php-fpm --nodaemonize --fpm-config usr/local/php/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
LimitNOFILE=1000000
LimitNPROC=1000000
LimitCORE=1000000

[Install]
WantedBy=multi-user.target
EOF

systemctl enable php-fpm
```

接着还需要添加 php-fpm 配置文件，并进行配置

```bash
cat > /usr/local/php/etc/php-fpm.conf << EOF
;;;;;;;;;;;;;;;;;;;;;
; FPM Configuration ;
;;;;;;;;;;;;;;;;;;;;;

;;;;;;;;;;;;;;;;;;
; Global Options ;
;;;;;;;;;;;;;;;;;;

[global]
pid = run/php-fpm.pid
error_log = log/php-fpm.log
log_level = warning

emergency_restart_threshold = 30
emergency_restart_interval = 60s
process_control_timeout = 5s
daemonize = yes

;;;;;;;;;;;;;;;;;;;;
; Pool Definitions ;
;;;;;;;;;;;;;;;;;;;;

[caddy]
listen = /dev/shm/php-cgi.sock
listen.backlog = -1
listen.allowed_clients = 127.0.0.1
listen.owner = caddy
listen.group = caddy
listen.mode = 0666
user = caddy
group = caddy

pm = dynamic
pm.max_children = 12
pm.start_servers = 8
pm.min_spare_servers = 6
pm.max_spare_servers = 12
pm.max_requests = 2048
pm.process_idle_timeout = 10s
request_terminate_timeout = 120
request_slowlog_timeout = 0

pm.status_path = /php-fpm_status
slowlog = var/log/slow.log
rlimit_files = 51200
rlimit_core = 0

catch_workers_output = yes
;env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
EOF
```

进行如下修改：

```bash
# 内存小于 3000M
pm.max_children = 内存（单位：M） /3/20
pm.start_servers = 内存（单位：M） /3/30
pm.min_spare_servers = 内存（单位：M） /3/40
pm.max_spare_servers = 内存（单位：M） /3/20

# 内存为 3000M - 4500M
pm.max_children = 50
pm.start_servers = 30
pm.min_spare_servers = 20
pm.max_spare_servers = 50

# 内存为 4500M - 6500M
pm.max_children = 60
pm.start_servers = 40
pm.min_spare_servers = 30
pm.max_spare_servers = 60

# 内存为 6500M - 8500M
pm.max_children = 70
pm.start_servers = 50
pm.min_spare_servers = 40
pm.max_spare_servers = 70

# 内存大于 8500M
pm.max_children = 80
pm.start_servers = 60
pm.min_spare_servers = 50
pm.max_spare_servers = 80
```

---

### 安装PHP 扩展
#### Redis 扩展

下载 Redis Server 和 PECL Redis

```bash
cd /usr/local/src
wget https://github.com/redis/redis/archive/7.2.4.tar.gz
wget https://pecl.php.net/get/redis-6.0.2.tgz
```

编译安装 Redis Srever

```bash
mkdir -p /usr/local/redis/{etc,var}
cd /usr/local/src
tar zxf 7.2.4.tar.gz
cd redis-7.2.4
make -j$(nproc)
make PREFIX=/usr/local/redis install
```

将 bin 目录下可执行文件进行链接并配置 redis

```bash
ln -s /usr/local/redis/bin/* /usr/local/bin/
cp /usr/local/src/redis-7.2.4/redis.conf /usr/local/redis/etc/
vim /usr/local/redis/etc/redis.conf

# 改成如下
pidfile /var/run/redis/redis.pid
logfile /usr/local/redis/var/redis.log
dir /usr/local/redis/var
daemonize yes
bind 127.0.0.1 ::1

# maxmemory 计算方法：内存（单位：M）/8*1000000
maxmemory 512000000
```

添加用户 并设置文件夹权限

```bash
useradd -M -s /sbin/nologin redis
chown -R redis:redis /usr/local/redis/{var,etc}
```

添加 Redis Server 启动脚本，并设置开机自启

```bash
cat > /usr/lib/systemd/system/redis-server.service << 'EOF'
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=forking
PIDFile=/var/run/redis/redis.pid
User=redis
Group=redis

Environment=statedir=/var/run/redis
PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p ${statedir}
ExecStartPre=/bin/chown -R redis:redis ${statedir}
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf
ExecStop=/bin/kill -s TERM $MAINPID
Restart=always
LimitNOFILE=1000000
LimitNPROC=1000000
LimitCORE=1000000

[Install]
WantedBy=multi-user.target
EOF

systemctl enable redis-server
systemctl start redis-server
```

接着安装 PECL Redis 并为 PHP 开启 PECL Redis 扩展

```bash
cd /usr/local/src
tar zxf redis-6.0.2.tgz
cd redis-6.0.2
/usr/local/php/bin/phpize
./configure --with-php-config=/usr/local/php/bin/php-config
make -j$(nproc)
make install

echo 'extension=redis.so' > /usr/local/php/etc/php.d/redis.ini
```

#### Memcached 扩展

首先需要下载 Memcached、libMemcached 和 PECL Memcached

```bash
cd /usr/local/src
wget https://memcached.org/files/memcached-1.6.24.tar.gz
wget https://launchpad.net/libmemcached/1.0/1.0.18/+download/libmemcached-1.0.18.tar.gz
wget https://pecl.php.net/get/memcached-3.2.0.tgz
```

开始编译安装 Memcached

```bash
# 添加用户
useradd -M -s /sbin/nologin memcached

cd /usr/local/src
tar zxf memcached-1.6.24.tar.gz
cd memcached-1.6.24
./configure --prefix=/usr/local/memcached
make -j$(nproc)
make install

# 链接可执行文件
ln -s /usr/local/memcached/bin/memcached /usr/bin/memcached
```

添加 Memcached 启动脚本，并设置开机自启

```bash
cat > /usr/lib/systemd/system/memcached.service << 'EOF'
[Unit]
Description=memcached daemon
After=network.target

[Service]
Environment=PORT=11211
Environment=USER=memcached
Environment=MAXCONN=1024
Environment=CACHESIZE=256
Environment="OPTIONS=-l 127.0.0.1"
ExecStart=/usr/bin/memcached -p ${PORT} -u ${USER} -m ${CACHESIZE} -c ${MAXCONN} $OPTIONS
PrivateTmp=true
ProtectSystem=full
NoNewPrivileges=true
PrivateDevices=true
CapabilityBoundingSet=CAP_SETGID CAP_SETUID CAP_SYS_RESOURCE
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

[Install]
WantedBy=multi-user.target
EOF

systemctl enable memcached
systemctl start memcached
```

在编译安装之前先给 libMencached 打个补丁

```bash
cd /usr/local/src
tar zxf libmemcached-1.0.18.tar.gz

cat > libmemcached-build.patch << 'EOF'
diff -up ./clients/memflush.cc.old ./clients/memflush.cc
--- ./clients/memflush.cc.old	2017-02-12 10:12:59.615209225 +0100
+++ ./clients/memflush.cc	2017-02-12 10:13:39.998382783 +0100
@@ -39,7 +39,7 @@ int main(int argc, char *argv[])
{
	options_parse(argc, argv);
 
-	if (opt_servers == false)
+	if (!opt_servers)
	{
		char *temp;
 
@@ -48,7 +48,7 @@ int main(int argc, char *argv[])
		opt_servers= strdup(temp);
	}
 
-	if (opt_servers == false)
+	if (!opt_servers)
    {
		std::cerr << "No Servers provided" << std::endl;
		exit(EXIT_FAILURE);
EOF

patch -d libmemcached-1.0.18 -p0 < libmemcached-build.patch
```

然后编译安装 libMencached

```bash
cd libmemcached-1.0.18
sed -i "s@lthread -pthread -pthreads@lthread -lpthread -pthreads@" ./configure
./configure --with-memcached=/usr/local/memcached
make -j$(nproc)
make install
```

接下来编译安装 PECL Memcached 并为 PHP 开启 Memcached 扩展

```bash
cd /usr/local/src
tar zxf memcached-3.2.0.tgz
cd memcached-3.2.0
/usr/local/php/bin/phpize
./configure --with-php-config=/usr/local/php/bin/php-config
make -j$(nproc)
make install


cat > /usr/local/php/etc/php.d/memcached.ini << EOF
extension=memcached.so
memcached.use_sasl=1
EOF
```

#### Imagick 扩展

下载 Imagemagick 和 PECL Imagick

```bash
cd /usr/local/src
wget https://imagemagick.org/archive/ImageMagick.tar.gz
wget https://pecl.php.net/get/imagick-3.7.0.tgz
```

编译安装 Imagemagick

```bash
cd /usr/local/src
tar zxf ImageMagick.tar.gz
cd ImageMagick-7.1.1-29
./configure --prefix=/usr/local/imagemagick \
--enable-shared \
--enable-static
make -j$(nproc)
make install
```

编译安装 PECL Imagick 并为 PHP 开启 Imagick 扩展

```bash
cd /usr/local/src
tar zxf imagick-3.7.0.tgz
cd imagick-3.7.0
/usr/local/php/bin/phpize
./configure --with-php-config=/usr/local/php/bin/php-config \
--with-imagick=/usr/local/imagemagick
make -j$(nproc)
make install
echo 'extension=imagick.so' > /usr/local/php/etc/php.d/imagick.ini
```

---

至此 Caddy2 + MariaDB + PHP 安装完成
