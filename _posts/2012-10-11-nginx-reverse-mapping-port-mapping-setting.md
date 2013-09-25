---
layout: post
title: nginx反向映射时设置端口映射
category : server
tags:
- nginx
- web server
- 反向代理
- 端口映射
published: true
---
该遇到一个很变态的问题，因为以前没遇到过，所以折腾了很久，现将折腾记录留下，以备查看。

遇到一个域名，默认居然不是80端口，比如说abc.com:7180。面对这个一个端口使用nginx做反向代理时，ngixn并不会同时反向设置端口，而是默认以80端口转发，这样就会出现后端request.getServerPort()端口获取的端口号是80，而不是7180，这种情况下就会请求失败，同时因为后端spingsecurity转发地址不支持绝对地址以及端口设置，所以唯一的方案就是在nginx里解决，找了很多文档后终于发现以下解决方案，经测试完美解决了我遇到的问题：
``` xml
  server {
    listen 7180;
    server_name abc.com;

    location / {
        proxy_pass http://10.1.1.1:7180;
        proxy_set_header Host $host:7180;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

```

以上唯一的修改之处就是在$host对象后面加上了:7180，即在nginx后端转发时手动加上我所需要的端口，解决了nginx反向代理是端口映射问题。
