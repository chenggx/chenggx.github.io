---
title: 使用LaraDock配置laravel-echo-sever遇到的坑
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/0eb30f2442a7d9337119f7dba74bd11372f001e0.jpg
top: false
categories: 开发环境
tags:
 - LaraDock
 - 开发环境
---

> 前情提要：项目需要使用 webSocket 进行通信，并且使用的是 [LaraDock](https://laradock.io/) 作为环境。所以使用 laravel-echo-server、laravel-horizon、php-worker 容器进行环境的部署。因为对 Docker 并不懂，仅仅只是使用 LaraDock，所以过程中遇到了很多问题，特此记录下来。

## 坑二 [参考资料](https://github.com/laradock/laradock/issues/1340#issuecomment-390484505)

> LaraDock 的 Nginx 使用多站点配置的时候，laravel-echo-server 无法链接到 web 进行认证。

通过在 nginx 容器配置中添加别名 ** laradock/docker-compose.yml **
```yml
### NGINX Server #########################################
    nginx:
      build:
        context: ./nginx
        args:
          - PHP_UPSTREAM_CONTAINER=${NGINX_PHP_UPSTREAM_CONTAINER}
          - PHP_UPSTREAM_PORT=${NGINX_PHP_UPSTREAM_PORT}
      volumes:
        - ${APP_CODE_PATH_HOST}:${APP_CODE_PATH_CONTAINER}
        - ${NGINX_HOST_LOG_PATH}:/var/log/nginx
        - ${NGINX_SITES_PATH}:/etc/nginx/sites-available
      ports:
        - "${NGINX_HOST_HTTP_PORT}:80"
        - "${NGINX_HOST_HTTPS_PORT}:443"
      depends_on:
        - php-fpm
      networks:
        # before ----
        # - frontend
        # - backend
        # end before ----
        frontend:
        backend:
            aliases:
              - my-site.com    ##别名地址，在laravel-echo-server.json 中填写该地址就可以了
```
更改完配置之后需要重新构建 nginx 和 laravel-echo-server 容器。
并重新开启。



## 坑二
> larave-echo-server 的 ssl 证书

1. 直接配置 ssl 证书。（不明白证书需要发在目录的什么位置，所以 google 了很多内容之后选择了下面的方法。以后如果有机会补充第一点）
2. 配置 nginx 反向代理。 [laravel-echo-server](https://github.com/tlaverdure/laravel-echo-server)
 
```nginx
#以下内容需要在你的 nginx 站点配置文件的 server{} 块之内
location /socket.io {
	    proxy_pass http://laravel-echo-server:6001; #填写非 https 的地址进行反向代理，如果 Echo 和 Nginx 在同一个服务器可以填写 localhost
	    proxy_http_version 1.1;
	    proxy_set_header Upgrade $http_upgrade;
	    proxy_set_header Connection "Upgrade";
	}
```
3. 配置 laravel-echo-server 配置文件 laravel-echo-server.json 为如下内容

```json
{
        "authHost": "https://laravel-echo-server", //注意：改地址指向 web 站点，即 echo-server 想web端认证的地址
        "authEndpoint": "/broadcasting/auth",
        "clients": [],
        "database": "redis",
        "databaseConfig": {
                "redis": {
                        "port": "6379",
                        "host": "redis"
                }
        },
        "devMode": true,
        "host": null,
        "port": "6001",
        "protocol": "http",
        "socketio": {},
        "sslCertPath": "",
        "sslKeyPath": ""
}
```

## 操作技巧

1. 不使用 -d 参数可以更好的进行调试
```shell
docker-compose up laravel-echo-server
```
2. 使用 logs 参数可以查看详情的日志输出
```shell
docker-compose logs laravel-echo-server
```
