---
title: "Git 多用户及仓库配置"
slug: "git_config"
date: 2022-11-06T11:56:29+08:00
lastmod: 2022-11-06T11:56:29+08:00
description: Git 多用户、仓库及使用代理的设置
tags:
- Git
categories:
- 学习
image: 
---

## 前言

在之前的 Hugo 博客系列文章有提到过一些基本的 Git 配置和操作，一般来说我们写 Hugo 博客只需要配置全局账号以及单个仓库即可。但凡事皆有例外，如果你需要多个 Git 用户，或者说需要配置多个不同的仓库，那么应该怎么设置呢？现在我们就来看看。

<!--more-->

---

## 配置设置

### 多仓库设置

如果只使用单个 Github/Gitee 账户来管理多个仓库的话，只需要针对每个本地仓库设置远程仓库 Remote 即可。例如，我在 Github 上面有两个项目，一个名为 demo ，另一个名为 example ：

- demo 仓库：

```bash
mkdir /path/to/demo # 创建仓库文件夹
cd /path/to/demo # 进入该仓库目录
git init # 指定 git 仓库并初始化
git remote add origin git@github.com:user/demo.git # 添加远程仓库地址
```

- example 仓库：

```bash
mkdir /path/to/example # 创建仓库文件夹
cd /path/to/example # 进入该仓库目录
git init # 指定 git 仓库并初始化
git remote add origin git@github.com:user/example.git # 添加远程仓库地址
```

像这样单账户，多仓库的，只需要为每个仓库设置远程仓库 Remote 即可。

### 多账户设置

如果你需要管理多个 Git  用户，应该怎么操作？继续看：

#### 创建密钥对

譬如，我有一个 Github 账号，一个 Gitee 账号，它们需要分开操作；首先打开 Git bash 进入 ~/.ssh 目录为它们分别创建密钥对：

首先创建 Github 密钥对：

```bash
cd ~/.ssh
ssh-keygen -t rsa -C "email" -f "github" # email 更换为你的 Github 邮箱，-f 参数为文件名，可自行设置
cat github.pub # 查看公钥内容
```

复制公钥内容，然后打开 Gituhb 添加刚才生成的公钥到 SSH Key

接着是 Gitee：

```bash
ssh-keygen -t rsa -C "email" -f "gitee" # email 更换为你的 Gitee 邮箱，-f 参数为文件名，可自行
cat gitee.pub # 查看公钥内容
```

同样，复制公钥内容，添加到 Gitee 的 SSH 公钥

然后添加私钥并进行测试

```bash
ssh-add ~/.ssh/github
ssh-add ~/.ssh/gitee
ssh -T git@github
ssh -T git@gitee
```

成功则会返回欢迎你的用户名信息。

#### 创建配置文件

创建完密钥对后，需要创建一个配置文件来分配私钥以及主机

```bash
touch config
vim config # 或者直接用 VS Code 编辑
```

config 配置文件内容如下：

```bash
# Github
Host github.com
  HostName github.com
  User username # username改成你的 Github 用户名
  IdentityFile ~/.ssh/github # 私钥位置

# Gitee
Host gitee.com
  HostName gitee.com
  User username # username改成你的 Gitee 用户名
  IdentityFile ~/.ssh/gitee # 私钥位置
```

#### 用户及 Remote 配置

由于拥有多个 Git 用户，就不能使用全局配置了；而是应该对仓库进行单独的用户配置，所以要先取消全局用户设置：

```bash
git config --global unset user.name username
git config --global unset user.email email
```

接着对本地仓库分别进行单独配置，假定我 Github 项目的本地仓库文件夹为 github ：

```bash
cd /path/to/github # 进入仓库目录
git config --local user.name username # 为该仓库设置用户
git config --local user.email email # 为该仓库设置邮箱
git remote remove origin # 移除之前的远程仓库地址
git remote add origin git@github.com:username/repositories.git # 添加远程仓库地址
```
这里注意下新的 remote 格式，因为之前设置了 config 文件，所以相对的地址为：git@(config 中的 Host):用户名/仓库名.git 即：`git@github.com:uesrname/repositories.git`

然后按照同样的方法为 Gitee 设置即可。

### 扩展配置

除了为 Git 分配主机和私钥外，也可以为自己的服务器进行设置，首先为服务器设置密钥登陆，可以参考 [这里](/p/setvps/#设置秘钥登录) ；然后将密钥对下载到本地 ~/.ssh 目录下，然后重命名并编辑 config 文件，如下：

```bash
# Github
Host github.com
  HostName github.com
  User username # username改成你的 Github 用户名
  IdentityFile ~/.ssh/github # 私钥位置

# Gitee
Host gitee.com
  HostName gitee.com
  User username # username改成你的 Github 用户名
  IdentityFile ~/.ssh/gitee # 私钥位置

# Tencent Cloud
Host tencent
  HostName xx.xx.xx.xx # 服务器 IP
  User root
  Port 22
  IdentityFile ~/.ssh/tencent # 私钥位置
```

之后就可以通过 `ssh tencent` 进行远程服务器的登陆了。

---

## 代理设置

由于 Github 服务器在海外，以及某些不可控因素，导致在推送及拉取代码时老是失败；我们可以为在 config 配置文件中为 Github 添加代理访问。打开 config 文件进行编辑：

> 使用前请确保你拥有 “应用” 

```bash
# Github
Host github.com
  HostName github.com
    ProxyCommand connect -S 127.0.0.1:10808 %h %p
  User username
  IdentityFile ~/.ssh/github
```

需要注意 `ProxyCommand connect -S 127.0.0.1:10808 %h %p` 这行：

> -S 代表 Socks，-H 代表 HTTP
> 
>  如果你按照我之前的文章架设的 “应用” ，则 127.0.0.1:10808 是 Socks 的端口，HTTP 的端口为 10809 

这样就可以通过代理来进行 Github 的推送/拉取操作，再也不用担心连接超时和连接失败啦。

## 总结

以上就是 Git 多用户、仓库配置及使用代理的全部过程啦，希望能给大家带来一定的帮助。有个问题就是我在创建 config 配置文件后无法通过 PS (Power Shell) 进行 SSH 连接；连接就报错 `Bad owner or permissions` ；网上也找了解决方案，但是都没有用。现在一直是使用 Terminal 打开 Git bash 进行 SSH 连接，以后有了好的解决方案会和大家进行分享；也希望能有懂的大佬给出一点建设性的意见，十分感谢🙏