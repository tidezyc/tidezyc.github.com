---
layout: post
title: Keepalived + nginx实现高可用性和负载均衡
category : server
published: true
tags: linux linux-ha ha keepalived nginx
---

在前面的一篇中讲到了Heartbeat作为高可用服务架构的解决方案，今天有试验了一种全新的解决方案，即采用Keepalived来实现这个功能。

Keepalived 是一种高性能的服务器高可用或热备解决方案，Keepalived可以用来防止服务器单点故障(单点故障是指一旦某一点出现故障就会导致整个系统架构的不可用)的发生，通过配合Nginx可以实现web前端服务的高可用。

Keepalived实现的基础是VRRP协议，Keepalived就是巧用VRRP协议来实现高可用性(HA)的.

VRRP(Virtual Router Redundancy Protocol)协议是用于实现路由器冗余的协议，VRRP协议将两台或多台路由器设备虚拟成一个设备，对外提供虚拟路由器IP(一个或多个)，而在路由器组内部，如果实际拥有这个对外IP的路由器如果工作正常的话就是MASTER，或者是通过算法选举产生，MASTER实现针对虚拟路由器IP的各种网络功能，如ARP请求，ICMP，以及数据的转发等；其他设备不拥有该IP，状态是BACKUP，除了接收MASTER的VRRP状态通告信息外，不执行对外的网络功能。当主机失效时，BACKUP将接管原先MASTER的网络功能。

VRRP协议使用多播数据来传输VRRP数据，VRRP数据使用特殊的虚拟源MAC地址发送数据而不是自身网卡的MAC地址，VRRP运行时只有MASTER路由器定时发送VRRP通告信息，表示MASTER工作正常以及虚拟路由器IP(组)，BACKUP只接收VRRP数据，不发送数据，如果一定时间内没有接收到MASTER的通告信息，各BACKUP将宣告自己成为MASTER，发送通告信息，重新进行MASTER选举状态。

1. 安装Keeplived依赖
-----

安装keepalived之前，也要安装一些依赖库

安装 openssl

> yum install openssl*

安装popt

> yum install popt*

安装ipvsadm
> yum isntall ipvsadm

安装libnl-dev
> yum install libnl-dev*


2. 安装Keepalived
------

keepalived安装包地址：

> http://www.keepalived.org/software/keepalived-1.2.7.tar.gz

下载解压后编译配置

> ./configure --prefix=/usr/local/keepalived


编译配置需要确保一下几项为Yes状态：

> Use IPVS Framework       : Yes
> IPVS sync daemon support : Yes
> IPVS use libnl           : Yes
> Use VRRP Framework       : Yes

然后就可以编译安装了：

> make && make install

因为没有使用keepalived的默认路径安装（默认是/usr/local）,安装完成之后，需要做一些工作

``` bash
cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
#复制keepalived启动文件到默认路径，也可以通过设置环境变量的path实现

cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
#复制服务启动脚本到，以便可以通过service控制keepalived服务

cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
#复制keepalived服务脚本到默认的地址，也通过修改init.d/keepalived文件中的相应配置实现

mkdir -p /etc/etc/keepalived/
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
vi /etc/keepalived/keepalived.conf
#复制默认配置文件到默认路径，其实也可以在/etc/init.d/keepalived中设置路径

chkconfig keepalived on
#开机启动服务
```

<!--more-->

3. 配置Keepalived
-------------

接下来就是配置了，很简单，直接上配置文件

先是主服务器：

``` bash
global_defs
{
notification_email    #通知email，根据实际情况配置
{
admin@example.com
}
notification_email_from admin@example.com
smtp_server 127.0.0.1
stmp_connect_timeout 30
router_id node1         #节点名标识，主要用于通知中
}

vrrp_instance VI_NODE {
  state MASTER          #配置为主服务器
  interface eth0        #通讯网卡
  virtual_router_id 100 #路由标识
  priority 200          #优先级，0-254
  advert_int 5          #通知间隔，实际部署时可以设置小一点，减少延时
  
  authentication {
    auth_type PASS
    auth_pass 123456    #验证密码，用于通讯主机间验证
  }

  virtual_ipaddress {
    192.168.1.206       #虚拟ip，可以定义多个
  }
}
```

接下是从服务器设置：

``` bash
global_defs {
  notification_email {
    admin@example.com
  }
  notification_email_from admin@example.com
  smtp_server 127.0.0.1
  stmp_connect_timeout 30
  router_id node2
}

vrrp_instance VI_NODE {
  state BACKUP           #与主服务器对应
  interface eth0         #从服务器的通信网卡
  virtual_router_id 100  #路由标识，和主服务器相同
  priority 100           #优先级，小于主服务器即可
  advert_int 5           #这里是接受通知间隔，与主服务器要设置相同
  
  authentication {
   auth_type PASS
    auth_pass 123456     #验证密码，与主服务器相同
  }
  
  virtual_ipaddress {
    192.168.1.206        #虚拟IP，也要和主服务器相同
  }
}
```

上面的设置是最基础的设置，实现的功能是如果主服务器的Keepalived停止服务（一般情况下服务器宕机），则将虚拟IP切换至从服务器，主服务器恢复后从新切换回主服务器。

但是很多情况下我们面临的处境是nginx挂掉了，而这个时候Keepalived就不能发挥作用，这时候就需要我们来改良下Keepalived了。通过向Keepalived添加一个自定义脚本来监控neginx的运行状态，如果nginx进程结束，则kill Keepalived进程，以此来达到主从服务器的切换功能。

我们在修改上面配置的主服务器的配置文件，在中间添加脚本实现

``` bash
global_defs {
   notification_email {
     admin@example.com
   }
   notification_email_from admin@example.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id nginx_master
}
vrrp_script chk_http_port {
   script "/usr/local/keepalived/nginx.sh"  #在这里添加脚本链接
   interval 3       #脚本执行间隔
   weight 2         #脚本结果导致的优先级变更
}
vrrp_instance VI_NODE {
    state MASTER
    interface eth0
    virtual_router_id 100
    priority 200
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    track_script {
        chk_http_port     #添加脚本执行
    }
    virtual_ipaddress {
        192.168.1.206
    }
}
```

具体的配置可以参考另一篇文章[Keepalived配置详解](http://thinkjet.me/)

如果我们使用了LVS+Keepalived集成，那么keepalived可以代替ipvsadm来配置LVS，可以方便的通过配置就可以搞定，这在另一篇文章[Keepalived+LVS配置详解](http://thinkjet.me/)

修改完配置文件我们写我们的上面配置的nginx.sh，当然我们假定Nginx已经安装完成

``` bash
#!/bin/bash
A=`ps -C nginx --no-header |wc -l`
if [ $A -eq 0 ];then
   killall keepalived
fi
```

上面的脚本简单的查看nginx进程是否存在，不存在就kill keepalived进程。

接下来我们对上面的哦脚本修改一下，当脚本检测到nginx没有运行的时候会尝试去启动nginx以此，如果失败则停掉keepalived进程

``` bash
#!/bin/bash
A=`ps -C nginx –no-header |wc -l`
if [ $A -eq 0 ];then
  /usr/local/nginx/sbin/nginx #nginx命令的路径
  sleep 3
  if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
    killall keepalived
  fi
fi
```

把脚本保存到上面配置的具体路径（我这里是/usr/local/keepalived）,然后很重要的一步是修改脚本的执行权限

> chmod +x nginx.sh

4. 运行Keepalived
------

配置完成后就可以运行看下效果了,分别在主从服务器上启动nginx和keepalived

>  service keepalived start

启动之后通过·ip a·命令查看主服务器的网络信息，可以看到在eth0网卡下生成了192.168.1.206这个虚拟ip，并可通过这个ip访问到nginx

然后我们关闭nginx的进程（如果配置了一次尝试重启那要注意下），然后我们可以通过`ps -e`查看keepalived进程是否关闭，正常情况下查看网络信息中，可以看到eth0网卡下的虚拟ip已经解除，然后在从服务器的网络信息中可以看到从服务器的eth0网卡绑定了虚拟ip，通过这个ip就访问到了从服务器的nginx去了，这是我们重新启动主服务器的nginx和keepalieved，我们可以发现虚拟ip就绑回到了主服务器。

这样就实现了基本双击主从热备功能了。

这里注意下防火墙的问题，就是这问题困扰了我很久。找了一些资料才将问题解决

因为Keepalived之间是通过组播来通知对方的是否存活，以及发送优先级的，并且通过组播来选举MASTER的，而224.0.0.18就是常用的组播地址，防火墙开启允许这个组播地址通信就可以了：

1.如果用的是默认防火墙,只需要添加：

> iptables -I RH-Firewall-1-INPUT -d 224.0.0.18 -j ACCEPT

2.如果是自己用脚本设置的防火墙，需要添加如下规则

> iptables -A OUTPUT -o eth0 -d 224.0.0.18 -j ACCEPT
> iptables -A OUTPUT -o eth0  -s 224.0.0.18 -j ACCEPT
> iptables -A INPUT -i eth0 -d 224.0.0.18 -j ACCEPT
> iptables -A INPUT -i eth0  -s 224.0.0.18 -j ACCEPT</pre>


5. 总结
------

+ keepalived通过虚拟路由实现双机热备，相比其他方案具有一定的优越性
+ 因为是固定主从热备，该方案比较适合两个互备服务器性能有差异的情况
+ Keepalived同样可以实现双主互备，通过设置互为主备，然后通过DNS负载均衡到不同vip就可以实现
