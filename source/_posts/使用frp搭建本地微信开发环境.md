---
title: 使用frp搭建本地微信开发环境
date: 2019-03-08 09:11:32
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/fhdfjsklfjsj12334jl432;.jpeg
top: false
categories: 开发环境
tags:
 - frp
 - 开发环境
---

> **2019-3-10** 更新记录

再使用 TP5 进行开发的时候需要注意一下几点
1. 关闭 trace 调试模式。
2. 在使用路由的时候尽量确定对应接口请求的类型，如果不知道类型建议使用 **Route::any()**;

*可以使用 [微信消息测试工具](http://www.fangbei.org/tool/message)进行测试*。

## 原因
之前在做相关微信开发时，因为需要验证公网可以访问的到的域名，所以总是将代码推到线上服务器进行相关的测试。每次有些小改动也要进本地代码推送到服务器，服务器拉取最新代码这种重复性的操作，感觉十分繁琐，一直想找一个简单方便的方式进行微信本地环境的开发。😓 无奈之前自己太懒了......总是拖。最近几天好好研究了下特此记录下来。

## 准备工作
1. 一台公网可以访问的服务器（或使用海外服务器）。
2. 一个已经备案的域名。
3. 内网穿透工具 [frp](https://github.com/fatedier/frp)

## 服务端配置

### 下载 frp
下载对应系统的 [frp](https://github.com/fatedier/frp/releases) 包。因为我的公网服务器是 centOS6-x64 所以下载了 `frp_0.24.1_linux_amd64.tar.gz` 包。

### 配置 frp 服务端

 1. 在服务器上解压下载好的 frp 包。

```bash
 tar zxvf frp_0.24.1_linux_amd64.tar.gz
 ```

2. 进入解压好的目录，编辑 frp 服务器端配置文件 **frps.ini**。

 ```ini
 [common]
 bind_port = 7000             #frp 绑定的端口
 vhost_http_port = 8000       #http 访问端口
 ```
 
 3. 启动服务端 frp
 
 ```bash
 ./frps -c ./frps.ini
 ```

## 解析域名
在域名服务商配置域名解析，将域名解析到上面的服务器 ip 地址上。

## 配置 nginx 反向代理进行 frp 端口的转发

1. 配置 nginx 

```nginx
server {
    listen 80;
    server_name wx.domain.com; # 绑定域名

    location / {
        proxy_pass http://localhost:8000; # 转发至本机8000,即在frps.ini中配置>的vhost_http_port
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

2. 重启 nginx


## 本地端配置
>  本地使用的是 vagrant 虚拟机。使用的同样是 centOS6-x64。在配置 frp 客户端之前需要将项目代码在 nginx 中正确配置，指定的域名就是上面解析的域名。

1. 根据本地系统环境下载对应的  [frp](https://github.com/fatedier/frp/releases) 包。
2. 编辑 frp 客户端配置文件 **frpc.ini**。

 ```ini
 [common]
 server_addr = x.x.x.x     #frps 所在的服务器的 IP
 server_port = 7000        #frps 绑定的端口
 use_encryption = true     #将 frpc 与 frps 之间的通信内容加密传输，将会有效防止流量被拦截。 
 use_compression = true     # 对传输内容进行压缩，可以有效减小 frpc 与 frps 之间的网络流量，加快流量转发速度，但是会额外消耗一些 cpu 资源。
 
 [web]
 type = http
 local_port = 80   #为本地机器上 web 服务对应的端口
 custom_domains = wx.domain.com   #上一步中解析好的域名
 ```


3. 启动 frp 客户端
 ```bash
 ./frpc -c ./frpc.ini
 ```

## 测试
如果一切正常那么现在随便找一台设备，访问刚才的域名就可以访问到本地的项目了。








