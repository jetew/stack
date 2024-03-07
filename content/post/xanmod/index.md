---
title: "Linux 安装 XanMod 内核开启 BBR V3"
slug: "xanmod"
description: "Linux 安装 XanMod 内核开启 BBR V3 使 VPS 降低延迟提升性能"
comments: false
date: 2024-03-01T05:26:41+08:00
lastmod: 2024-03-01T05:26:41+08:00
draft: false
toc: true
image: ""
tags: ["Linux"]
---

## 前言

### 关于 XanMod 内核

- [XanMod 官网](https://xanmod.org/)

默认的 Linux 内核被设计为一种通用解决方案，能够在不同的系统和硬件配置上提供广泛的兼容性。它稳定、可靠且经过广泛测试，但并不总是针对特定用例提供最佳性能。

自定义内核（例如 XanMod）则能满足这一需求。XanMod 内核是基于最新稳定版本的 Linux 内核，旨在通过低延迟提高系统的响应性能。它是由社区驱动的项目，结合了其他内核的最佳特性和独特的增强功能，更加专注于优化桌面、多媒体和游戏工作负载，以提供更具响应性和流畅性的 Linux 使用体验。

### 注意事项

XanMod 内核目前仅支持 X86 结构的 CPU，且目前仅支持 Debian/Ubuntu

### 内核选择

XanMod 项目提供多种不同的内核构建，每种构建都针对特定的用例和硬件配置。

#### XanMod MAIN 内核

MAIN 内核是标准的 XanMod，包括最新稳定版本的 Linux 内核，并针对桌面、多媒体和游戏工作负载进行了优化。MAIN 内核有四个版本可供选择：

- `linux-xanmod-x64v1`
- `linux-xanmod-x64v2`
- `linux-xanmod-x64v3`
- `linux-xanmod-x64v4`

#### XanMod EDGE 内核

EDGE 内核专为想要最新功能和增强的用户而设计，它们包括最近版本的 Linux 内核，并针对高性能工作负载进行了优化。EDGE 内核有三个版本可供选择：

- `linux-xanmod-edge-x64v2`
- `linux-xanmod-edge-x64v3`
- `linux-xanmod-edge-x64v4`

#### XanMod LTS 内核

LTS（长期支持）内核是为将稳定性和可靠性放在优先考虑的用户而设计，它们包括较旧但经过更多测试的 Linux 内核版本，并针对通用工作负载进行了优化。LTS 内核有四个版本可供选择：

- `linux-xanmod-lts-x64v1`
- `linux-xanmod-lts-x64v2`
- `linux-xanmod-lts-x64v3`
- `linux-xanmod-lts-x64v4`

#### XanMod RT 内核

RT（实时）内核是为关键应用场景设计的，例如 Linux 游戏服务器、流媒体、直播制作和超低延迟需求的用户，它们包括 PREEMPT_RT 实时补丁，可降低系统的延迟并提高响应性。RT 内核有三个版本可供选择：

- `linux-xanmod-rt-x64v2`
- `linux-xanmod-rt-x64v3`
- `linux-xanmod-rt-x64v4`

这些特定的 XanMod 内核构建被设计用于特定的硬件配置，涵盖从较旧的 x86-64 系统到最新的 AMD 和 Intel 处理器。您可以在 [XanMod 官网](https://xanmod.org) 上找到不同内核构建硬件兼容性的更详细信息。

---

## 安装步骤

### 准备工作

#### 安装必要组件

```bash
apt update
apt install -y gnupg
```

#### 检测 CPU 支持版本

XanMod 有各种版本，需依据 CPU ISA（指令集架构）而选择合适的版本，我们可以通过官方提供的脚本来确认：

```bash
awk -f <(wget -O - https://dl.xanmod.org/check_x86-64_psabi.sh)
```

输出结果：

```bash
CPU supports x86-64-v4
```

这里可以看到我的 CPU 是支持 v4 版本的，安装时可以按照此结果进行选择。

**注意**：一定要选择符合的版本进行安装，否则将导致无法正常启动！

#### 安装 XanMod

注册 PGP 密钥：

```bash
wget -qO - https://dl.xanmod.org/archive.key | gpg --dearmor -o /usr/share/keyrings/xanmod-archive-keyring.gpg
```

添加存储库：

```bash
echo 'deb [signed-by=/usr/share/keyrings/xanmod-archive-keyring.gpg] http://deb.xanmod.org releases main' | tee /etc/apt/sources.list.d/xanmod-release.list
```

更新 APT 软件包：

```bash
apt update
```

安装 XanMod 内核，这里我以 XanMod MAIN x64 v4 内核为例，你可以根据自己的需求进行安装 MAIN、EDGE、LTS 或 RT 内核：

```bash
apt install -y linux-xanmod-x64v4
```

安装完成后可通过 `reboot` 进行重启使内核生效，如果遇到通过命令重启不生效的情况，可以去云服务商后台进行重启操作。

#### 检测内核

在终端执行 `uname -r` 查看内核，输出结果：

```bash
6.7.6-x64v4-xanmod1
```

这说明已经成功安装并切换至 XanMod 内核了。

#### BBR3 状态与队列算法

在终端执行以下命令开启 BBR3 并设置队列算法：

```bash
cat > /etc/sysctl.conf << EOF
net.ipv4.tcp_congestion_control=bbr
net.core.default_qdisc=fq_pie
EOF
```

然后执行 `sysctl -p` 看到以下内容则说明开启成功：

```bash
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq_pie
```

然后执行 `reboot` 重启生效。

下面是我抄来的内核参数配置：

```bash
cat > /etc/sysctl.conf << EOF
vm.swappiness = 1
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq_pie
fs.file-max = 1000000
fs.inotify.max_user_instances = 8192
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.route.gc_timeout = 100
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_synack_retries = 1
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 32768
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_max_orphans = 32768
EOF
```

在终端执行以下命令检查 BBR3 状态：

```bash
modinfo tcp_bbr
```

输出内容：

```bash
name:           tcp_bbr
filename:       (builtin)
version:        3
description:    TCP BBR (Bottleneck Bandwidth and RTT)
license:        Dual BSD/GPL
file:           net/ipv4/tcp_bbr
author:         David Morley <morleyd@google.com>
author:         Arjun Roy <arjunroy@google.com>
author:         Kevin Yang <yyd@google.com>
author:         Yousuk Seung <ysseung@google.com>
author:         Priyaranjan Jha <priyarjha@google.com>
author:         Soheil Hassas Yeganeh <soheil@google.com>
author:         Yuchung Cheng <ycheng@google.com>
author:         Neal Cardwell <ncardwell@google.com>
author:         Van Jacobson <vanj@google.com>
```

可以看到已成功开启 BBR3，如果报错可以先执行 `depmod`

执行以下命令查看队列算法：

```bash
sysctl net.core.default_qdisc
```

输出结果：

```bash
net.core.default_qdisc = fq_pie
```