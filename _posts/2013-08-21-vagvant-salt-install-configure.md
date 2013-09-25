---
layout: post
title: 基于Vagrant+Salt的集成环境搭建
category : Linux
tags: linux 运维 vagrant salt saltstack
published: true
---

### 为什么是Vagrant和salt

Vagrant是什么？如果简单点说就是一个像非常流行的windows系统的ghost镜像一样的东西，Vagrant再原有的系统镜像的基础上，加入了自己的系统配置，成了一个经过了配置的系统，并以.box文将的形式保存。当然不同于ghost镜像，Vagrant的box是专用于vm的，具有端口映射，文件映射等功能，更关键的是vagrant的所有的操作都可以命令行完成，而且很多操作都可以一句命令就搞定，非常之快。
正因为这样的好处，vagrant在团队开放中可以发挥非常大的作用，团队可以生成一个集成了团队所需的开发测试环境的box，然后分发给团队所有人使用，既免除了团队成员每个人都去自建测试环境的繁琐，而且更重要的是保证了团队成员测试环境的完全统一。包括测试环境的修改、变更，都可以通过修改box镜像，然后分发完成，省去了成员单独更新带来的各种问题。

除了对于团队协作的便利高效，Vagrant对于个人使用也是异常方便，除了端口映射和文件映射这样的贴心功能，Vagrant更大的优势在于可重现的实现，比如昨天我们在测试环境里测试了应用，产生了测试记录和错误，今天我们需要出重新测试一下，需要恢复初始的系统状态，一般情况下，我们需要删除现有vm，然后重新安装vm，然后重新配置，但是有了vagrant就不需要这么麻烦了，直接重新初始化vagrant就可以了，直接可以恢复到初始化环境，这对于可重复性测试实在是太方便了。

所以，我们对Vagrant的简单定义就是一个使用VirtualBox动态创建和配置轻量级的，可重现的，可分享的，便携的虚拟机环境。当然除了virtualbox，Vagrant现在已经支持vmware等更多的虚拟环境实现。

Salt全称是SaltStack，是一个集中配置管理系统，如果你知道Puppet或者chef，那么就是salt就是类似puppet的东西。并且相比puppet而言，salt更加的简洁，更易于安装使用和扩展。

如果你不了解puppet，那么也没关系，我们就来看看salt解决什么实际的问题。

假设我们手上有10台服务器，现在每台服务器都需要我们去配置下防火墙然后安装个jdk环境，这个要怎么搞呢，当然最简单的办法就是一台一台搞过来，问题是这样真的简单吗，如果我们有100台服务器呢，一万台呢？这个时候就是salt发挥作用的时候了，通过salt，我们可以批量配置我们的服务器，可能需要简单的一条命令，我们就完成了原来一天都不一定能搞定的几百台服务器的配置工作。

当然除了批量配置，salt的作用还不止这些，通过将系统配置写进配置文件，然后通过git等版本管理工具，我们可以追溯服务器的每次配置变动，并可以实现配置的重复性实现或者回滚实现，保证了系统管理的可追溯性和可重复性。


### 安装VirtualBox

Vagrant简单点理解是对virtualbox或vmware等虚拟环境的封装，所以第一步当然是安装虚拟化支持环境，这里我们以virtualbox为例介绍vagrant的安装过程。

使用vmware可以参考官方文档 [Vagrant with Vmware][1]

Virtualbox可以安装参考官方教程 [http://virtualbox.org][2] 进行安装，基本支持现在主流的操作系统，不同的系统的安装方式也不一样，安装时注意版本的选择，更多说明可以参考我的另一篇文章： [Linux安装VirtualBox][3]

### 安装Vagrant

旧版的Vagrant需要安装ruby等支持环境，而新版的vagrant已经支持一键安装方式安装，非常方便高效。

 1. 首先下载vagrant的安装文件：[http://downloads.vagrantup.com][4]
 2. 直接安装即可，安装过程将自动添加 `vagrant` 到系统环境变量中
 3. 安装完成后通过在终端中输入vagrant命令来验证安装

#### 添加vm box

Vagrant的box就是对VirtualBox的封装，封装之后为.box类型的文件，我们可以从 [http://www.vagrantbox.es][5] 下载所需的box文件，当然我们也可以通过 [veewee](https://github.com/jedi4ever/veewee)这个工具自己从iso镜像开始自己创建一个box。在这里我们还是用一个现有box:

` http://developer.nrel.gov/downloads/vagrant-boxes/CentOS-6.4-i386-v20130427.box`

我们可以先下载到本地，可以直接通过以下命令添加box到vagrant

`  vagrant box add centos64_32  http://developer.nrel.gov/downloads/vagrant-boxes/CentOS-6.4-i386-v20130427.box`

这里 ` centos64_32 ` 是box的别称，如果不设置就采用默认的名称 ` base ` ,当然推荐还是设置一个可读性的名字，特别是在添加了多个box的情况下。

我们也可以选择先下载box到本地，然后添加到vagrant：

` vagrant box add centos64_32_local /home/johnny/centos.box `

其中` /home/johnny/centos.box ` 是我们下载的box的本地地址。

#### 初始化 box

调用 ` vagrant init centos64_32 ` 命令初始化一个 vagrant box，该命令会在当前目录下初始化一个`Vagrantfile`文件，这是vagrant的标准配置文件，包括了box文件、网络映射、端口映射以及文件映射等配置，一般我们不需要修改什么配置直接启动即可：

` vagrant up `

在启动过程中，如果遇到警告`  your guest additions are out of date ` 这主要是box的additions 与 virtualbox的additions版本不一致造成的，忽略这个警告可能会导致虚拟系统和宿主系统间的网络通信和文件映射出现文件。

Addtions是guest系统和host系统进行通信和分享的通道，vagrant的主要功能都需要addtions的支持，更新Addtions可以具体参考virtualbox的官方方法[https://www.virtualbox.org/manual/ch04.html][6]。

### 安装Salt

Salt 是一主多从的结构，如果是多台服务器，可以选择一台作为主，其他作为从，如果只有一台机器的话可以同时安装主从服务，或者选择无主模式，简单的安装可以通过bootstrap安装。

` wget -O - http://bootstrap.saltstack.org | sudo sh `

或者也可以通过yum进行安装（需要EPEL库）

` yum install salt-minion`

` yum install salt-master `

### 整合Vagrant和Salt

要在Vagrant上安装Salt除了可以进入vm后手动安装Salt之外，还有一种更好的选择是使用[salty-vagrant][7] 插件，该插件是Vagrant的一个插件，本身是一个Ruby gem，为Vagrant提供了Salt Stack的支持。

> Vagrant 原生支持puppet和chef，大概salt太新，还没来得及支持

#### 安装 vagrant-salt 插件

可以通过以下命令安装salt-vagrant插件：

` vagrant plugin install vagrant-salt `

安装完之后需要修改`Vagrantfile`文件，添加salt的支持：

``` bash
Vagrant.configure("2") do |config|
  ## Choose your base box
  config.vm.box = "centos64_32"

  ## For masterless, mount your salt file root
  config.vm.synced_folder "salt/roots/", "/srv/salt/"

  ## Use all the defaults:
  config.vm.provision :salt do |salt|

    salt.minion_config = "salt/minion"
    salt.run_highstate = true

  end
end
```
+ 该例子使用了无主模式，其他模式设置已参考vagrant-salt的文档配置
+ ` config.vm.synced_folder "salt/roots/", "/srv/salt/" `将本地当前目录下的salt/roots目录映射到了guest系统的/srv/salt目录
+ ` salt.run_highstate = true ` 在vagrant up的时候调用`state.highstate`，该命令让所有的minion获取走自己的SLS定义，然后在本地调用对应的state module（user，pkg，service等）来达到SLS描述的状态。
+ 无主模式下需要设置`file_client: local`
+ 无主模式的主要区别在于调用命令采用了` salt-call `,而不是`salt`。

#### 运行 Vagrant with Salt

运行`vagrant up`就可以简单的启动我们的guest系统。

然后运命`vagrant ssh`就可以进入系统

其他的一些命令：
`vagrant halt` 关机
`vagrant suspend` 休眠
`vagrant destroy` 销毁当前系统
`vagrant reload` 重启系统
`vagrant package` 将当前的系统重新打包成一个新的box文件

### 总结

Vagrant加上Salt的整合方案对于提高开发效率和团队协作方面都表现出了非常高的效率，特别对于现在流行的软件可持续交付方面的实现提供了非常好的解决方案以及非常大的想象空间。

  [1]: http://www.vagrantup.com/vmware
  [2]: http://virtualbox.org/
  [3]: http://thinkjet.me/linux-install-virtualbox.html
  [4]: http://downloads.vagrantup.com/
  [5]: http://www.vagrantbox.es/
  [6]: https://www.virtualbox.org/manual/ch04.html
  [7]: https://github.com/saltstack/salty-vagrant