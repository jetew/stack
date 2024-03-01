---
title: "我写博客的方式"
slug: "how_to_write"
date: 2022-11-01T13:18:02+08:00
lastmod: 2022-11-01T13:18:02+08:00
description: 分享一下我日常是怎么在不同终端进行博客写作与数据同步的
tags:
- Hugo
- Git
- Github
- Blog
image: 
---

## 前言

一般来说，大家会有一些想法或者突如其来的灵感需要进行记录；那么我们会选择将它们记录在手机便签、文本编辑器或者直接用笔写下来；等最后再梳理成文。正常情况下我们都是在电脑前进行写作，但是有时候不方便在电脑前进行写作；那应该怎么办？今天就来看下我的日常写作方式。

<!--more-->

看过前面文章的应该知道，我自己是将 Hugo 博客源码通过 Git 推送到 Github 远程仓库然后由 Github Actions 自动部署到服务器上面，所以我写文章的流程就是通过 Git 同步远程仓库，写作完成后再推送到远程仓库通过 Github Actions 自动部署就可以了。那么现在就一起来看下这个过程的实现。

我的话，平常使用的客户端为：家里电脑 (WindowsX2) 、公司电脑 (Windows) 、笔记本 (Linux)、手机 (Android)；简化一点就是 Windows、Linux、Android。一般情况下，我们就是 Win 和 Android 了。要实现在安卓使用 Git 的话，我们就需要安装一款神级软件 **Termux** 

---

## 介绍及安装

### Termux

- [Termux 官网](https://termux.dev/)

- [Github 项目地址](https://github.com/termux)

> Termux 是一款 Android 终端模拟器和 Linux 环境应用程序，无需 root 或设置即可直接运行。 自动安装最小的基本系统 - 可以使用 APT 或其他包管理器。

你可以通过 [F-Droid](https://f-droid.org/en/packages/com.termux/) 或者 [Github](https://github.com/termux/termux-app/releases) 进行下载，下载完成之后进行更换清华源，安装 Git ，创建目录软链接，安装 Hugo 即可。

> 无法下载或者下载慢可以找国内下载地址。
> 
> Termux 安装使用基于国光大佬的 [Termux 高级终端安装使用配置教程](https://www.sqlsec.com/2018/05/termux.html) ；写的非常全，强烈推荐观看！！！

#### 更换清华源

打开 **Termux** 输入如下命令：

```bash
sed -i 's@^\(deb.*stable main\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/termux-packages-24 stable main@' $PREFIX/etc/apt/sources.list
sed -i 's@^\(deb.*games stable\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/game-packages-24 games stable@' $PREFIX/etc/apt/sources.list.d/game.list
sed -i 's@^\(deb.*science stable\)$@#\1\ndeb https://mirrors.tuna.tsinghua.edu.cn/termux/science-packages-24 science stable@' $PREFIX/etc/apt/sources.list.d/science.list
pkg update
```

安装基础工具：

```bash
pkg update
pkg install -y curl wget
```

#### 安装 Git

进行 Git 安装：

```bash
pkg update
pkg install -y git
```

#### 创建目录软链接

为了方便管理，我们使用文件管理器在手机存储根目录下创建 **termux** 文件夹，然后进行软链接：

```bash
termux-setup-storage
```

执行以上命令，会弹出授权窗口，确认授权后 Termux 就可以访问 SD 卡文件，然后为创建的 termux 目录创建软链接：

```bash
ln -s /data/data/com.termux/files/home/storage/shared/termux termux
```

执行完上述命令后，就会在 **~** 目录下创建 termux 快捷方式，此目录是链接到我们刚才创建的 **termux** 目录，方便进行管理。

#### 安装 Hugo

```bash
pkg update
pkg install -y hugo
```

上述操作都完成后，就可以在 Android 端使用 Hugo 了。

---

## Git 配置

### 全局账户配置

在使用 Git 之前需要进行账户配置，让 Github 知道你是谁：

```bash
git config --global user.name username # username为你的 Github 用户名
git config --global user.email xx@email.com # XX@email.com为你注册 Github 的邮箱
```

> 注意，这里是全局设置，也就是说你所有的本地仓库推送时都是使用这个用户和邮箱

### 添加密钥对

键入如下命令创建密钥对：

```bash
ssh-keygen -t ed25519 -C "email"
```

接下来一路回车即可。

### 公钥添加至 Github

输入 `cat ~/.ssh/id_ed25519.pub` 查看公钥内容，然后复制；打开 Github 点击： **右上角头像 - Settings - SSH and GPG keys - New SSH Key** 把刚才复制的公钥粘贴到 **Key** 中，命名随意，然后保存。

> 详细图文过程可以参考 [这里](/archives/hugo/#添加密钥)


### 本地仓库配置

因为我们创建了一个 Termux 软链接，为了方便管理，我将 Hugo 博客仓库放在此目录中，打开 **Termux** 进入到 **termux** 目录，创建博客目录，并设置为 Git 项目仓库：

```bash
cd termux
mkdir -p hugo/blog(项目名，我以 blog 为例)
cd blog
git init
```

这时候打开文件管理器，进入 termux 目录，可以看到里面已经创建了 hugo/blog 目录，这就是我们的 Hugo 博客仓库了。

由于 Github 现在默认的分支是 main 所以我也设置本地分支为 main：

```bash
git branch -m main
```

这个时候可能会提示添加到安全仓库：

```bash
Initialized empty Git repository in /storage/emulated/0/termux/hugo/blog/.git/
➜  blog git branch -m main
fatal: detected dubious ownership in repository at '/storage/emulated/0/termux/hugo/blog/'
To add an exception for this directory, call:

        git config --global --add safe.directory /storage/emulated/0/termux/hugo/blog
```

按照提示输入命令添加到安全仓库即可：

```bash
git config --global --add safe.directory /storage/emulated/0/termux/hugo/blog
```

添加完成后再更换分支名：

```bash
git branch -m main
```

### 添加远程仓库

```bash
git remote add origin git@github.com:username/repositories.git # username换成你的github用户名，repositories更换为你的仓库名
```

### 拉取远程仓库

输入下面命令将远程仓库的代码拉取到本地：

```bash
git pull origin main
```

拉取完成之后，还需要将我们添加的主题 clone 下来，因为之前时通过 `git submodule` 进行链接的；比如我在用的 **zzo** 主题：

```bash
git clone https://github.com/zzossig/hugo-theme-zzo.git themes/zzo
```

完成之后，我们就可以通过 Android 端进行写作了。

---

## 本地预览及推送

### 本地预览

在 Android 端写作完成后，我们也可以通过 `hugo server` 命令进行本地预览，但是在 termux 下不太一样，直接运行 `hugo server` 会报错，所以要加上一个 `noBuildLock` 参数：

```bash
hugo server --noBuildLock
```

通过上述命令，即可在 Android 端通过 **localhost:1313** 进行本地预览，同理，如果你要通过 `hugo` 生成静态文件，也是需要添加 `noBuildLock` 参数。

### 推送

在写作完成及预览无误后，就可以进行推送到远程仓库了。与桌面端一样，通过 `push` 进行推送：

```bash
git add .
git commit -m "xxxx"
git push origin main
```

推送完成后，等待 Github Actions 自动部署完成后，即可进行访问。

---

## 总结

以上就是我在不同客户端通过 Git 进行同步数据、写作的过程了，希望能给大家提供一个参考。需要注意的是：一定要记得，在不同的客户端下，第一件事是通过 `git pull` 将远程仓库拉取同步到本地仓库，然后再进行写作

## 参考资料

1. [Termux 高级终端安装使用配置教程](https://www.sqlsec.com/2018/05/termux.html)

2. [通过 SSH 连接到 GitHub](https://docs.github.com/cn/authentication/connecting-to-github-with-ssh)