+++
title = 'Frp内网穿透参数设置'
date = 2024-04-22T15:48:22+08:00
draft = false
+++

*原发布于2021年4月22日*
http://blog.gensou.cc/archives/669

Frp内网穿透的教程各大博客都能找到，但具体参数很少有讲清楚的。这里个人对已学习的部分参数做笔记以供参考。这里的需求是一台内网设备映射多个端口至一台公网设备。

### 服务端（公网设备）
在Frp穿透中，公网设备是服务端，配置在frps.ini中设置
```ini
[common]
bind_port = 7000
vhost_http_port = 8080
```

**[common]**

- bind_port是必须设置的端口，这个参数的端口在公网设备使用、用来在公网设备和内网设备沟通。

- vhost_http_port是当有http映射时，如果设置，作为默认的端口，这个参数的端口在公网设备使用、用来其他任意客户端打开映射的网站时访问的端口，即访问：公网设备IP:vhost_http_port。

需要开放的端口

- bind_port（这里为7000）

- 所有其他客户端访问公网ip时需要访问的端口，包括公网设备的frps.ini中的vhost_http_port和内网设备的frpc.ini中的所有remote_port（后面介绍）。

### 客户端（内网设备）
在Frp穿透中，内网设备是客户端，配置在frpc.ini中设置
```ini
[common]
server_addr = 106.x.x.x
server_port = 7000

[web]
type = http
local_ip = 127.0.0.1
local_port = 9091
custom_domains = 106.x.x.x

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 8081
```

**[common]**

- server_addr填写公网设备的IP。

- server_port填写公网设备用于与内网设备沟通的端口，即和服务端的frps.ini的bind_port相同。

**[web]**

- type是映射类型，这里需要映射网站，填写http。
在http映射中，需要设置本地架设网站的ip和端口（即在本机浏览器上访问网站的ip和端口），这个端口是网站后端监听的端口（这里例子为9091），而其他客户端要访问网站，需要的是外网设备ip和映射端口（这里例子为106.x.x.x:8080），这个端口需要在本地开放。

- local_ip是网页IP，通常是127.0.0.1，即本机。

- local_port是网页端口。

- custom_domains是网页的域名，必须填写；如果没有域名，可以填写外网设备的IP。

**[ssh]**

- type是映射类型，这里需要映射ssh，填写tcp。

- local_ip是本地SSH的IP，通常是127.0.0.1，即本机。

- local_port是本地SSH的端口，通常是22。

- remote_port是本地该端口映射到外网设备后，其他客户端需要访问本地时需要访问的端口，即其他客户端连接ssh实际需要访问的端口（这里例子为106.x.x.x:8081），这个端口需要在外网设备开放。