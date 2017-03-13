---
title: 建站伊始：使用 rsync 和 git 两种方式部署 Hexo
---
心血来潮，想有个小站记录一下学习过程中的各种收获，于是便有了这个地方，这篇文章就记录一下搭建本站过程中遇到的一些问题。

# 概述

本站使用 [Hexo](https://hexo.io/) 搭建，其是一个轻量级的博客系统，允许用户使用 `Markdown` 来撰写博客，并在本地渲染生成一个静态网站，再通过多种可选的方式发布到远程服务器上。

本文主要针对部署过程进行记录，对发布之前的步骤都不再进行过多描述，详细的使用姿势可以参考它的[官方文档](https://hexo.io/docs/)（[中文版](https://hexo.io/zh-cn/docs/)）。

# 部署

## 免密登录服务器

不管是使用 `rsync` 或 `git` 来部署，我们都一定不会希望在执行自动化部署脚本的时候被命令行索要远程服务器登录密码的提示而打断，而不得不去自己的某个密码管理工具或者印象笔记中寻找这个该死的密码。避免这个令人沮丧且枯燥无味的过程的唯一办法，就是将自己的一个`公钥`配置到要登录的远程用户的`置信密钥列表`之内，然后在 `ssh` 的时候指定使用对应的`私钥`，就可以愉快的跳过这个过程了。

### 本地生成 `ssh` 密钥

为了达到这个目的，我们首先在本机创建一对新的公私密钥（当然如果你已经有一对可以使用的密钥了，也可以跳过这一步）：

```bash
# -C 后面实际跟的是备注，可以有选择的输入任何能帮助你分辨这个密钥的内容
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ~/.ssh/my_blog
```

以上命令在路径 `~/.ssh` 下生成了一对密钥 `my_blog` 和 `my_blog.pub`，其中 `my_blog.pub` 是公钥。

### 服务器端生成新用户

登录远程服务器，执行以下命令：

```bash
# 添加名称为 'git' 的用户
# --disabled-password：仅可以使用密码之外的认证方式登录
sudo adduser --gecos[^1] 'git version control' --disabled-password git
```

> Debian/Ubuntu 下有两个相似的命令 `useradd` 和 `adduser`，两者的差别在与：
> 使用 `useradd` 创建的用户，如果没有通过参数来配置各种选项的，是不会在 `/home` 下添加新用户的主目录的，并且默认的 shell 是 `/bin/sh`，即系统的默认值。
> 如果是使用 `adduser` 来创建用户，那系统会进入交互式状态，循序渐进的引导用户创建新用户。

因为我的服务器是通过 `Nginx` 来进行反向代理的，`Nginx` 运行的用户组是 `www-data`，因此需要把新用户加入该用户组以保证 `git 用户`对网站有操作权限：

```bash
usermod -aG www-data git
```

接着把之前在本机生成的密钥 `my_blog.pub` 中的内容复制到 `/home/git/.ssh/authorized_keys` 中：

```bash
mkdir -p /home/git/.ssh
touch /home/git/.ssh/authorized_keys
chmod 600 /home/git/.ssh/authorized_keys
chmod 700 /home/git/.ssh
```

### 测试登录

至此，应该可以通过下面的命令顺利的登录到远程服务器了：

```bash
ssh -i ~/.ssh/my_blog git@<your_remote_ip>
```

在此基础之上，就可以使用 `rsync` 和 `git` 两种方式来部署博客了。

## 通过 `rsync` 部署

### 本机

本机首先安装 [hexo-deployer-rsync](https://github.com/hexojs/hexo-deployer-rsync)

```bash
npm i hexo-deployer-rsync --S
```

并在 `_config.yml` 中添加[配置](https://hexo.io/docs/deployment.html#Rsync)：

```yaml
deploy:
    type: rsync
    host: <your_remote_ip>
    user: git
    root: /var/www/blog
```

### 服务器端

服务器端则需要确保安装了 `rsync`：

```bash
apt-get install -y rsync
```

## 通过 `git` 部署

### 本机

本机首先安装 [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)：

```bash
npm i hexo-deployer-git --S
```

并在 `_config.yml` 中添加[配置](https://hexo.io/docs/deployment.html#Git)：

```yaml
deploy:
    type: git
    repo: ssh://git@<your_remote_ip>:<port>/home/git/blog.git
```

这里会将博客同步至 `/home/git/blog.git` 目录下。

### 服务器端

在服务器端添加 `git` 仓库：

```bash
mkdir /home/git/blog.git
cd /home/git/blog.git
git init --bare
```

并且由于我的 `Nginx` 是将网站的根目录指向 `/var/www/blog` 的，因此还需要配置一个 `hooks` 脚本，使博客有更新时能自动的将文件拷贝至该目录下：

```bash
# /home/git/blog.git/hooks/post-update
#!/bin/bash -l
GIT_REPO=/home/git/blog.git
TMP_GIT_CLONE=/tmp/blog
PUBLIC_WWW=/var/www/blog
rm -rf ${TMP_GIT_CLONE}
git clone $GIT_REPO $TMP_GIT_CLONE
rm -rf ${PUBLIC_WWW}/*
cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}
```

再修改一下相关文件的权限，保证 `git 用户`有权限执行相关命令：

```bash
chmod +x /home/git/blog.git/hooks/post-update
chown git:git /home/git/ -R
```

## 部署博客

通过以上两种方式中的任意一种完成配置之后，就可以在本机使用 `hexo d -g` 来部署博客了。

----

参考：

- https://liuzhichao.com/2015/hello-hexo.html


[^1]: [Gecos_field](https://en.wikipedia.org/wiki/Gecos_field) is typically used to record general information about the account or its user(s) such as their real name and phone number.