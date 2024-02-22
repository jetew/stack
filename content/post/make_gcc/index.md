---
title: "CentOS 7 编译安装 gcc"
slug: "make_gcc"
date: 2022-12-10T15:43:06+08:00
lastmod: 2022-12-10T15:43:06+08:00
description: 在 CentOS 7 下编译安装新版 gcc
tags:
- Linux
categories:
- 学习
image:
---

## 前言

前两天折腾手上的 CentOS7 服务器，在编译安装软件时发现编译失败；查看了下日志和文档发现是因为 gcc 和 g++ 版本过低了。而通过 yum 查找发现只有 4.8 版本，于是决定通过编译安装新版本，历经了重重困难后终于是安装完毕，并在此进行记录。

<!--more-->

---

## 安装步骤

### 准备工作

1. 首先安装任意版本 gcc 和一些需要的软件：

```bash
yum update
yum install -y gcc gcc-c++ m4 bzip2 screen
```

2. 下载 gcc 源码

通过 [官方仓库](https://gcc.gnu.org/pub/gcc/releases) 下载或 [镜像地址](https://gcc.gnu.org/mirrors.html) 寻找服务器就近地址；找到需要安装的版本下载并解压，我这里选的 9.5.0：

```bash
mkdir /opt/gcc
cd /opt/gcc
wget https://gcc.gnu.org/pub/gcc/releases/gcc-9.5.0/gcc-9.5.0.tar.gz
tar zxvf gcc-9.5.0.tar.gz
```

1. 下载依赖库

gcc 官网提供了依赖库下载地址，但是一个个下载比较麻烦，可以通过 gcc 源码自带的 `download_prerequisites` 文件；它可以自动下载关联的依赖库并且放置到 gcc 目录下，与 gcc 一起编译：

```bash
cd gcc-9.5.0
./contrib/download_prerequisites
```

执行后会自动下载依赖库并解压至当前目录下。

### 编译安装

完成上述操作后即可开始编译安装 gcc 了，首先进行 configure 配置与检查：

```bash
mkdir build && cd build
../configure --prefix=/usr/local/gcc --enable-threads=posix --disable-checking --disable-multilib --enable-languages=c,c++

# 输出
......
......
configure: creating ./config.status
config.status: creating Makefile
```

检查完毕生成 MakeFile 文件后，就可以进行编译安装了，时间比较长；为了防止网络等因素导致断连，可以创建一个窗口然后进行编译安装：

```bash
screen -S gcc
make && make install
```

这里可以通过 `make -j` 设置线程数来提升编译速度，如果不清楚该设置多少，可以通过 `make -j$(nproc)` 自动计算线程数：

```bash
make -j$(nproc) && make install
```

### 版本替换

1. 卸载旧版本

```bash
yum remove -y gcc gcc-c++
```

2. 链接新版本

```bash
ln -s /usr/local/gcc/bin/c++ /usr/bin/c++
ln -s /usr/local/gcc/bin/gcc /usr/bin/cc
ln -s /usr/local/gcc/bin/g++ /usr/bin/g++
ln -s /usr/local/gcc/bin/gcc /usr/bin/gcc
```

3. 添加环境变量

```bash
vim /etc/profile
```

修改 `/etc/profile` 文件，在末尾添加如下两句：

```bash
LD_LIBRARY_PATH=/usr/local/gcc/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH
```

然后刷新一下变量环境：

```bash
source /etc/profile
```

### 更新动态库

更新完成后需要对旧版本的动态库进行替换，首先查找一下文件：

```bash
find / -name "libstdc++.so"

# 输出
/usr/lib64/libstdc++.so.6
/usr/lib64/libstdc++.so.6.0.19
/usr/share/gdb/auto-load/usr/lib64/libstdc++.so.6.0.19-gdb.py
/usr/share/gdb/auto-load/usr/lib64/libstdc++.so.6.0.19-gdb.pyc
/usr/share/gdb/auto-load/usr/lib64/libstdc++.so.6.0.19-gdb.pyo
/usr/local/gcc/lib64/libstdc++.so.6.0.28
/usr/local/gcc/lib64/libstdc++.so.6
/usr/local/gcc/lib64/libstdc++.so
/usr/local/gcc/lib64/libstdc++.so.6.0.28-gdb.py
```

可以看到现在老版本是 `6.0.19` 而 `libstdc++.so.6` 则是 `libstdc++.so.6.0.19` 的软链接：

```bash
ls -l /usr/lib64/libstdc++.so.6

# 输出
lrwxrwxrwx. 1 root root 30 Dec 10 06:50 /usr/lib64/libstdc++.so.6 -> /usr/lib64/libstdc++.so.6.0.19
```

然后将 gcc 中的动态库复制过去，删除旧版本链接并重新链接新版本：

```bash
cp /usr/local/gcc/lib64/libstdc++.so.6.0.28 /usr/lib64/
rm -f /usr/lib64/libstdc++.so.6
ln -s /usr/lib64/libstdc++.so.6.0.28 /usr/lib64/libstdc++.so.6
```

更新完毕，接下来就可以正常使用了。

### 添加忽略（可选）

为防止误操作或运行自动化脚本情况导致被覆盖，可以将 cmake 添加忽略，编辑 `/etc/yum.conf` 

```bash
vim /etc/yum.conf
```

在末尾添加上：

```bash
exclude=gcc*
```

另外需要注意运行自动化脚本前查看一下是否会对配置文件进行编辑，若存在则需要修改脚本。
