---
title: "Linux 服务器设置 Swap 分区"
slug: "setswap"
date: 2022-11-11T19:15:56+08:00
lastmod: 2022-11-11T19:15:56+08:00
description: 小内存服务器设置 Swap 分区，解决内存不足的尴尬
comments: false
tags:
- Linux
- VPS
image: 
---

## 前言

这两天闲着没事，将手头上闲置的一台 512M 小内存服务器拿出来折腾，编译安装 MariaDB 的时候由于内存不足导致编译失败。由于我是个小白，且一直是使用的 2/4G 服务器，以前从来没有注意过这个；直到现在遇到这样的问题。。。

<!--more-->

### Swap 分区

别名：交换分区。 Swap 分区在系统的物理内存不够用的时候，把硬盘内存中的一部分空间释放出来，以供当前运行的程序使用。那些被释放的空间可能来自一些很长时间没有什么操作的程序，这些被释放的空间被临时保存到 Swap 分区中，等到那些程序要运行时，再从 Swap 分区中恢复保存的数据到内存中。

---

## 设置步骤

一般来说，很多服务器开通的时候就自带了 Swap 分区，在日常使用下并不需要对其进行额外的操作；但是有时候还是要视情况对 Swap 分区进行适当的增减；下面来看看 Swap 分区的设置 (这里我以我 4G 内存服务器作为演示，请根据实际情况修改数据) ：

### 查看分区

1. 通过 `free -h` 命令查看

```bash
free -h
# 输出
        total   used    free    shared    buff/cache    available
Mem:    3.6Gi   123Mi   3.3Gi   1.0Mi     182Mi         3.3Gi
Swap:   0B      0B      0B
```

通过以上命令可以看到当前内存及 Swap 分区使用状态，如果 Swap 分区和我这里一样显示 0B  0B  0B 则代表并没有开启 Swap 分区。

2. 通过 `swapon --show` 命令查看

```bash
swapon --show
```

若开启了 Swap 分区，则会显示 Swap 分区状态；若没有开启 Swap 分区，则不会返回数据。

### 创建分区

#### 交换分区的大小分配

查找了一下资料，发现众说纷纭。。。最后决定采用如下方案：

|**物理内存**|**建议 Swap 分区大小**|
|    :---:   |         :---:        |
|< 4GB       |2倍内存，不超过4G      |
|4-8G        |等于内存大小           |
|8-64G       |8G                    |
|64-256G     |16G                   |

#### 创建交换分区

1. 使用 `fallocate` 命令创建用于交换的文件

```bash
fallocate -l 4G /swapfile
```

如果报错可以使用 `dd` 命令：

```bash
dd if=/dev/zero of=/swapfile bs=1M count=4096
```

2. 设置权限

```bash
chmod 600 /swapfile
```

3. 设置交换分区

``` bash
mkswap /swapfile
# 输出
Setting up swapspace version 1, size = 4 GiB (4294963200 bytes)
no label, UUID=e3edf2fd-7259-4d00-98be-5afc604c1271
```

成功则返回如上信息。

4. 启用分区

```bash
swapon /swapfile
```

5. 验证分区

```bash
swapon --show
# 输出
NAME      TYPE SIZE USED PRIO
/swapfile file   4G   0B   -2
```

开启成功则返回 Swap 分区状态

---

### 开机启动及分配控制

#### 设置开机启动

1. 编辑 **/etc/fstab** 文件添加这一行：

```bash
vim /etc/fstab
# 添加如下:
/swapfile swap swap defaults 0 0
```

2. 或执行如下命令直接写入：

```bash
echo "/swapfile swap swap defaults 0 0" >>/etc/fstab
```

#### Swap 分区分配控制

- Swappiness

Swappiness 是一个 Linux 内核属性，用于定义系统使用交换空间的频率。 Swappiness 可以具有0到100之间的值。较低的值将使内核尽可能避免交换，而较高的值将使内核更积极地使用交换空间；默认的 swappiness 值为60.可以通过键入以下命令来检查当前的 swappiness 值：

```bash
cat /proc/sys/vm/swappiness
# 输出
60
```

对于桌面端，可以使用60的 swappiness 值；但对于生产服务器，可能需要设置较低的值，比如我设置为10：

1. 临时设置：

```bash
sysctl vm.swappiness=10
```

2. 要使之重启后保持不变，可以附加到 **/etc/sysctl.conf** 文件：

```bash
vim /etc/sysctl.conf
# 在最后添加如下
vm.swappiness=10
```

或者：

```bash
echo "vm.swappiness=10" >>/etc/sysctl.conf
```

---

### 删除及调整大小

如果需要对交换分区进行删除，可以按照下列步骤：

1. 停用交换分区

```bash
swapoff /swapfile
```

2. 在 **/etc/fstab** 文件中删除交换文件条目：

```bash
vim /etc/fstab
# 删除或注释下面一行：
/swapfile swap swap defaults 0 0
```

3. 删除 swapfile 文件

```bash
rm /swapfile
```

若需要修改分区大小，可先停用交换分区，然后直接删除 swapfile 文件并重新创建。

---

## 总结

以上就是本篇的全部内容，希望对大家有所帮助。

对于 Linux，无论是多大内存，还是要设立 Swap 交换分区，这样有利于在内存耗尽时及时启用 Swap 空间。另外，可以视情况对 Swap 分区大小进行临时调整，比如我前文所说用 512M 内存服务器编译安装高版本 MariaDB ，那么我可以先将 Swap 分区临时设置为 2G ，编译安装完成后再调整。
