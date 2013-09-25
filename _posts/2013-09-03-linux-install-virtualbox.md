---
layout: post
title: linux安装VirtualBox
category: Linux
tags: linux virtualbox
published: true
---

VirtualBox是一个开源的跨平台虚拟化软件，它可以安装在任何操作系统上，并让您同一台计算机上安装和运行多个客户操作系统。例如，如果你将它安装在你的Linux系统，可以运行Windows XP操作系统下的客户机操作系统或运行Linux操作系统在你的Windows系统等。这样，您就可以安装和运行许多来宾操作系统，只要你喜欢，唯一的限制是磁盘空间和内存。

#添加VirtualBox安装源

我们将通过VirtualBox官方提供的yum安装源来安装VirtualBox
这里以fedora系统为例，支持12～19的所有fedora版本

``` bash
## Fedora 19,18,17,16,15,14,13,12 ##
# cd /etc/yum.repos.d/
# wget http://download.virtualbox.org/virtualbox/rpm/fedora/virtualbox.repo
```
#安装依赖包
VirtualBox使用vboxdrv内核模块来控制和执行guest操作系统分配物理内存。没有这个模块，你仍然可以使用VirtualBox的创建和配置虚拟机，但他们不会工作。因此，为了使VirtualBox的全功能，您将需要先更新系统，然后安装一些额外的模块，如DKMS内核头文件和内核开发和一些依赖包。

``` bash
# yum update
# yum install binutils qt gcc make patch libgomp glibc-headers glibc-devel kernel-headers kernel-devel dkms
```

#安装VirtualBox
接下来我们就可以安装最新版的VirtualBox了
可以通过搜索命令查看最新的VirtualBox版本信息

``` bash
# yum search VirtualBox 
```

当前最新版是4.2版本

``` bash
# yum install VirtualBox-4.2
```
#重建内核模块

下面的命令将自动创建vboxusers组和用户，还搜索和自动重建所需的内核模块。如果下面的构建过程中失败了，你会得到一个警告消息。请看看/var/log/vbox-install.log记录跟踪构建过程中失败的原因。

``` bash
# /etc/init.d/vboxdrv setup
```

#启动VirtualBox

可以直接通过命令启动

``` bash
# VirtualBox
```

#可能遇到的问题
如果你得到像`KERN_DIR`这样的错误信息，或者提示构建过程中没有自动检测你的内核源代码目录这样的错误，可以通过使用以下命令。

``` bash
## RHEL / CentOS / Fedora 替换成自己的内核版本
KERN_DIR=/usr/src/kernels/2.6.18-194.11.1.el5-x86_64

## Export KERN_DIR ##
export KERN_DIR
```
