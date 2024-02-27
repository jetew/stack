---
title: "CentOS7 编译安装 Node"
slug: "make_node"
description: "在 CentOS 7 下编译安装新版 Node"
date: 2024-02-27T23:37:17+08:00
lastmod: 2024-02-27T23:37:17+08:00
draft: false
toc: true
image: ""
categories: ["学习"]
tags: ["Linux"]
---

## 前言

在之前的文章 [《Waline 服务端独立部署方案》](/archives/waline_deploy/) 中有讲到过进行源码编译安装 Node ，但只是带过一下。其实我在使用 Debian 时直接进行编译安装是没有问题的，但是在 CentOS7 下，遇到许多问题；于是便在此记录一下方便日后进行查阅。

---

## 安装步骤

### 准备工作

1. 软件依赖

  - `gcc` `g++` ≥ 10.1
  - GNU Make ≥ 3.81
  - Python

2. 安装依赖

```bash
yum install -y python3 make python3-pip
```

`gcc` 和 `gcc-c++` 由于 **yum** 源版本过低，需要另外进行编译安装，参考 [《CentOS 7 编译安装 gcc》](/archives/make_gcc/)

3. 下载源码

从 [Nodejs官网](https://nodejs.org/) 下载源码并进行解压

```bash
cd /usr/local/src
wget https://nodejs.org/dist/v20.11.1/node-v20.11.1.tar.gz
tar zxvf node-v20.11.1.tar.gz
```

### 编译安装

```bash
cd node-v20.11.1
./configure --prefix=/usr/local/node

# 输出
……
INFO: configure completed successfully

# 创建窗口，防止网络问题断开终端导致安装中止
screen -S node
make
make install
```

这里可以通过 `make -j` 设置线程数来提升编译速度，如果不清楚该设置多少，可以通过 `make -j$(nproc)` 自动计算线程数：

```bash
make -j$(nproc)
make install
```

为软件设置环境变量：

```bash
vim ~/.bashrc
```

在最后添加：

```bash
export PATH=/usr/local/node/bin:$PATH
export CPATH=/usr/local/node/include:$CPATH
export LD_LIBRARY_PATH=/usr/local/node/lib:$LD_LIBRARY_PATH
```

刷新变量环境：

```bash
source ~/.bashrc
```

完成后可通过 `node -v` 、 `npm version` 、 `npx -v` 进行验证，返回版本号则表示成功。


- 可能出现的问题

安装过程中可能会出现 `…… : Error: no such instruction: ……` 若遇到此问题则需要通过编译安装新版 `binutils`

#### 编译安装 binutils

可通过 [GNU官方镜像](https://ftp.gnu.org/gnu/binutils/) 下载新版源码，国内服务器可使用 [清华大学镜像](https://mirrors.tuna.tsinghua.edu.cn/gnu/binutils/)

```bash
cd /usr/local/src

# 国内服务器可更换为清华大学镜像下载地址
wget https://ftp.gnu.org/gnu/binutils/binutils-2.42.tar.gz
tar zxvf binutils-2.42.tar.gz
cd binutils-2.42
./configure --prefix=/usr/local/binutils

# 输出
…………
…………
configure: creating ./config.status
config.status: creating Makefile

make
make install
```

卸载旧版本：

```bash
yum remove -y binutils
```

设置环境变量：

```bash
vim ~/.bashrc
```

在最后添加：

```bash
export PATH=/usr/local/binutils/bin:$PATH
export CPATH=/usr/local/binutils/include:$CPATH
export LD_LIBRARY_PATH=/usr/local/binutils/lib:$LD_LIBRARY_PATH
```

刷新变量环境：

```bash
source ~/.bashrc
```

最后编辑 `/etc/yum.conf`  为 `node` 和 `binutils` 添加忽略：

```bash
vim /etc/yum.conf
```

在末尾添加上：

```bash
exclude=node*,binutils*
```

---

## 参考资料

1. [Building Node.js from source on supported platforms](https://github.com/nodejs/node/blob/main/BUILDING.md#building-nodejs-on-supported-platforms)

2. [guide to porting the binutils](https://sourceware.org/binutils/binutils-porting-guide.txt)