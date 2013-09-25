---
layout: post
title: Windows下设置nginx随系统启动
category : server
tags:
- nginx
- service
- web server
published: true
---
nginx是一个强劲的web服务器，很多站点其实都是基于nginx实现的，一般情况下nginx都是免安装的，一旦遇到服务器重启等情况就必须手动重新启动，这就比较麻烦，下面就讲一下怎么讲nginx安装为系统服务，并实现随系统启动：

1、下载微软的两个工具instsrv.exe和srvany.exe（[下载地址](http://thinkjet.me/wp-content/uploads/2012/10/nginx-service.rar)）

2、解压到nginx的目录下，用命令行进入到nginx目录，运行：

` instsrv Nginx D:\nginx\srvany.exe `
>Nginx为服务名

>D:\nginx\为nginx根目录

这样就安装了一个名为Nginx的服务，在这个过程中我们将srvany.exe注册成一个服务，而实际上srvany只是其注册程序的服务外壳，这样在系统启动时srvany.exe会随系统启动，加下去我们就要设置让nginx.exe随着srvany.exe一起启动。

通过配置Nginx服务的运行参数就可以实现nginx随srvany一起启动。

在上面下载的压缩包中还有nginx.reg文件，该文件实现通过修改注册表修改Nginx运行参数的功能

用记事本打开nginx.reg文件

``` bash

Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\NGINX\Parameters]
"Application"="D:\\nginx\\nginx.exe"
"AppParameters"=""
"AppDirectory"="D:\\nginx\\"

```

修改其中的nginx目录为你当前的nginx所在目录，修改完成后双击导入注册表
完成后就可以在服务中看到Nginx服务了，并通过设置自动形式实现nginx的开机启动了。
