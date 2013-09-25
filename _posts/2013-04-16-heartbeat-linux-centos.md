---
layout: post
title: Heartbeat实现集群高可用热备
published: true
tags: ha-linux heartbeat linux centos
---

公司最近需要针对服务器实现热可用热备，这几天也一直在琢磨这个方面的东西,今天做了一些Heartbeat方面的工作，在此记录下来，给需要的人以参考。

Heartbeat 项目是 Linux-HA 工程的一个组成部分，它实现了一个高可用集群系统。通过Heartbeat我们可以实现双机热备，以实现服务的持续性。

heartbeat （Linux-HA）的工作原理：heartbeat最核心的包括两个部分，心跳监测部分和资源接管部分，心跳监测可以通过网络链路和串口进行，而且支持冗 余链路，它们之间相互发送报文来告诉对方自己当前的状态，如果在指定的时间内未受到对方发送的报文，那么就认为对方失效，这时需启动资源接管模块来接管运 行在对方主机上的资源或者服务。

在 CentOS 中包含了该组件，可以直接Yum 进行安装，这里就不在赘述，本文意提供的是用编译安装方式安装heartbeat的步骤和配置项，以供参考。

安装Heartbeat对硬件没什么特别要求，主机间通信方式可以选择网络或者串口方式实现，如果选择通过网络链路来通信的就需要每个主机装有两张网卡。

本文选择安装的heartbeat版本为Heartbeat 3，Heartbeat 3与 2.x的最大差别在于，3 按模块把的原来2.x 拆分为多个子项目，并且提供了一个cluster-glue的组件，专用于Local ResourceManager 的管理。即heartbeat + cluster-glue + resouce-agent 三部分，

1、安装依赖包
--------

Heartbeat 所需的依赖包包括了：gcc gcc-c++ autoconf automake libnet libtool glib2-devel libxml2-devel bzip2-devel e2fsprogs-devel libxslt-devel libtool-ltdl-devel make wget docbook-dtds docbook-style-xsl

在centos下可以通过yum快速安装

``` bash
 	yum install -y gcc gcc-c++ autoconf automake libnet libtool glib2-devel libxml2-devel bzip2-devel e2fsprogs-devel libxslt-devel libtool-ltdl-devel make wget docbook-dtds docbook-style-xsl
```


2. 添加组和用户
------

``` bash
	groupadd haclient
	useradd -g haclient hacluster -M -s /sbin/nologin
```

3. 安装Cluster-Glue
--------

Cluster-Glue是一个heartbeat的组件，相当于一个中间层，可以将heartbeat和crm（pacemaker）联系起来，它主要包含2个部分，LRM和STONITH；

安装地址：http://hg.linux-ha.org/glue/archive/glue-1.0.9.tar.bz2

``` bash
	//下载解压后
	./autogen.sh
	./configure --prefix=/usr/local/heartbeat --sysconfdir=/etc/heartbeat libdir=/usr/local/heartbeat/lib64 LIBS='/lib64/libuuid.so.1'
	make && make install
```

* 32位环境需要将配置参数中的lib64 更改为 lib
* 安装过程中会从 sourceforge 下载一些文件，确保sourceforge可以正常访问

4. 安装Resource Agents
---------------

就是各种的资源的ocf脚本，这些脚本将被LRM调用从而实现各种资源启动、停止、监控等等

安装地址：https://github.com/ClusterLabs/resource-agents/tarball/v3.9.2
	
``` bash
	//下载解压后
	./autogen.sh
	./configure --prefix=/usr/local/heartbeat --sysconfdir=/etc/heartbeat libdir=/usr/local/heartbeat/lib64 CFLAGS=-I/usr/local/heartbeat/include LDFLAGS=-L/usr/local/heartbeat/lib64 LIBS='/lib64/libuuid.so.1'
	ln -s /usr/local/heartbeat/lib64/* /lib64/
	//建立一个软连接，避免编译时找不到所需要的包
	make && make install
```

* 32位环境需要将配置参数中的lib64 更改为 lib

<!--more-->

5.安装Heartbeat
-----
	
安装地址：http://hg.linux-ha.org/heartbeat-STABLE_3_0/archive/7e3a82377fa8.tar.bz2

``` bash
	//下载解压后
	./bootstrap
	./configure --prefix=/usr/local/heartbeat --sysconfdir=/etc/heartbeat CFLAGS=-I/usr/local/heartbeat/include LDFLAGS=-L/usr/local/heartbeat/lib64 LIBS='/lib64/libuuid.so.1'
	vi /usr/local/heartbeat/include/heartbeat/glue_config.h
	// 删除 glue_config.h 最后一行定义的配置文件路径，避免编译时产生的路径重复定义错误,Shift+g 跳到末行，dd删除
	// define HA_HBCONF_DIR "/usr/local/heartbeat/etc/ha.d/" :wq保存完成.
	make && make install
```

将配置文件复制到 /etc/heartbeat/ 下，并使用sed 修改路径

``` bash
	cp doc/ha.cf /etc/heartbeat/ha.d/
	cp doc/haresources /etc/heartbeat/ha.d/
	cp doc/authkeys /etc/heartbeat/ha.d/
	chkconfig --add heartbeat
	chkconfig heartbeat on
	chmod 600 /etc/heartbeat/ha.d/authkeys

	sed -i 's#/usr/lib/ocf#/usr/local/heartbeat/usr/lib/ocf#g' /etc/heartbeat/ha.d/shellfuncs
	sed -i 's#/usr/lib/ocf#/usr/local/heartbeat/usr/lib/ocf#g' /etc/heartbeat/ha.d/resource.d/hto-mapfuncs
	sed -i 's#/usr/lib/ocf#/usr/local/heartbeat/usr/lib/ocf#g' /usr/local/heartbeat/usr/lib/ocf/lib/heartbeat/ocf-shellfuncs
```

建立Resource-Agent 的脚本软连接，避免Heartbeat 找不到路径而无法工作

``` bash
	ln -s /usr/local/heartbeat/usr/lib/ocf /usr/lib/ocf
```

系统服务可以通过 service heartbeat start/stop 来启动停止，但启动之前先要进行一些配置

6. 进行配置（只简单说明下提供参考）
------

1. 主机名：可以通过`uname -n`命令查看当前主机名
假设现在两个主机分别为node1和node2

2. 设置主机ip：
	* node1: 
		* eth0:192.168.1.101(对外ip)
		* eth1:10.10.10.1(心跳ip)

	* node2 
		* eth0:192.168.1.102(对外ip)
		* eth1:10.10.10.2(心跳ip)

	* 虚拟ip:192.168.103
3. 编辑Heartbeat 检测参数配置文件，以下文件作为参考

> vi /etc/heartbeat/ha.d/ha.cf

下面是针对node1的配置，node2类似配置即可

``` bash
# ====配置文件内容开始====
 debugfile /var/log/ha-debug
 # 用于记录heartbeat的调试信息

logfile /var/log/ha-log
 # 用于记录heartbeat的日志信息

logfacility local0
 #用于log记录的记录

keepalive 2
 # 设置心跳间隔

deadtime 30
 # 在30秒后宣布节点死亡

warntime 10
 # 在日志中发出“late heartbeat“警告之前等待的时间，单位为秒

initdead 120
 # 网络启动时间

udpport 694
 # 广播/单播通讯使用的udp端口

#baud 19200
 #serial /dev/ttyS0
 # 使用串口heartbeat

bcast eth1
 # 使用网卡eth1广播发送心跳检测,也可以以多播(mcast)和单薄方式(ucast)发送心跳检查

auto_failback on
 # 当主节点从故障中恢复时,将自动切换到主节点

watchdog /dev/watchdog
 # 该指令是用于设置看门狗定时器，如果节点一分钟内都没有心跳，那么节点将重新启动

node node1
node node2
 # 集群中机器的主机名，与“uname –n”的输出相同。

ping 10.10.10.2
 # ping 网关或路由器来检测链路正常，这里设置node2的心跳检测地址

respawn hacluster /usr/local/heartbeat/lib64/heartbeat/ipfail
 # respawn调用 ipfail 来主动进行切换

apiauth ipfail gid=haclient uid=hacluster
 # 设置启动ipfail的用户和组
 # ====文件配置内容结束======
```

4. 编辑资源文件

> vi /etc/heartbeat/ha.d/haresources

``` bash
HA-01 192.168.0.1 mysqld
 #机器名 虚拟服务器IP  系统服务
```

*在启动的时候,节点node1启用192.168.1.101 的虚拟IP地址,对外提供服务，同时也启动 mysqld 服务,当关闭的时候, heartbeat停止mysqld 服务，然后取消ip地址. 这里的主机用是用命令 uname -n 获取到的一致,资源文件支持很多种配置方法，请参考其它说明文

5. 配置验证密钥

> vi /etc/heartbeat/ha.d/authkeys

``` bash
auth 1 	//认证序号1
1 crc   //序号1 crc
```

*规则说明 auth 后面填写序号，可任意填写，但第二行开头必须为序号名，然后为验证方式，支持三种( crc md5 sha1 )方式验证，最后面是自定义密钥

``` bash
chmod 600 /etc/heartbeat/ha.d/authkeys
//文件权限更改为600
```

以同样方式配置了node2之后，就可以启动heartbeat进入工作了

正常工作时，使用 ifconfig 查看网络连接信息，可看到Heartbeat 在 eth0 接口上新添加了一个虚拟IP地址

``` bash
[root@node1 opt]# ifconfig
eth0      Link encap:Ethernet  HWaddr 00:0C:29:EB:E3:5E
          inet addr:192.168.1.101  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::20c:29ff:feeb:e35e/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:21944 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8325 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2564223 (2.4 MiB)  TX bytes:1670947 (1.5 MiB)

eth0:1    Link encap:Ethernet  HWaddr 00:0C:29:EB:E3:5E
          inet addr:192.168.1.103  Bcast:192.168.1.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)
eth1      Link encap:Ethernet  HWaddr B8:AC:6F:38:13:E5
          inet addr:10.10.10.1  Bcast:10.10.10.255  Mask:255.255.255.0
          inet6 addr: fe80::baac:6fff:fe38:13e5/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:60926 errors:0 dropped:0 overruns:0 frame:0
          TX packets:92730 errors:2 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:8333807 (7.9 MiB)  TX bytes:15275228 (14.5 MiB)
          Interrupt:16
```

当节点出现故障时，本机会尝试关闭服务和虚拟IP, 同时另一台机器检测到故障会添加相同的虚拟IP地址，来对外提供服务，实现业务的不中断，

参考资源：

* http://www.oschina.net/p/heartbeat
* http://www.linux-ha.org/wiki/Heartbeat
