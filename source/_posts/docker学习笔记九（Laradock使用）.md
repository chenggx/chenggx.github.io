---
title: docker 学习笔记（九）
date: 2020-08-30 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/blog/2020/08/31/19-16-47-7b7fa6ae2465aeaaf3cf9e709874da3a-d457ee.jpg

tags:
  - docker
---

## 克隆 laradock 到本地

```bash
$ cd ~
$ git clone https://github.com/Laradock/laradock.git
$ cd laradock
$ git checkout -b v11.0
```

## 在 laradock 同级创建 wwwroot 目录作为网站主目录

```bash
$ mkdir ~/wwwroot
```

## 复制 laradock 项目中的 env-example 到当前目录并改名为 .env

```bash
$ cp env-example .env
```

## 编辑 .env

**在该配置文件中可以修改各种容器的配置，例如 mysql 密码、php 版本等，大家可以自行参考**

一下内容是需要修改的地方

```
# 设置网站主目录
APP_CODE_PATH_HOST=../wwwroot

# 开启 api 源镜像（嘿嘿，这就是开源软件的好处，我们可以给项目提交 pr，让项目可以兼容我国的网络）
CHANGE_SOURCE=true

# 设置 composer 镜像地址
WORKSPACE_COMPOSER_REPO_PACKAGIST=https://mirrors.aliyun.com/composer

# 设置 npm 镜像地址
WORKSPACE_NPM_REGISTRY=https://registry.npm.taobao.org
```

## 启动

启动我们需要等容器，然后就是耐心的等待了

```bash
$ docker-compose up -d nginx mysql redis workspace
```

## 完成

当看到如下内容就表示启动成功了

```bash
Creating laradock_mysql_1            ... done
Creating laradock_docker-in-docker_1 ... done
Creating laradock_redis_1            ... done
Creating laradock_workspace_1        ... done
Creating laradock_php-fpm_1          ... done
Creating laradock_nginx_1            ... done
```

## 创建 Laravel 项目 

接下来让我们看下 laradock 有什么优势吧

### 创建一个 laravel 项目（我们使用 learnku 的电商实战项目进行演示）

```bash
$ cd ~/wwwroot
$ git clone -b L05_7.x https://github.com/summerblue/laravel-shop.git
```

### 进入 workspace 容器配置项目

```bash
$ docker-compose exec workspace bash

workspace# cd laravel-shop
workspace# composer install
workspace# cp .env.example .env
workspace# php artisan key:generate
workspace# vim .env    //修改数据库部分，内容如下。
workspace# php artisan migrate
workspace# php artisan db:seed
```
![env 配置](http://static.xiangdangnian.net.cn/20200907115535.png)

**查看 laradock 中的 .env 文件，获取数据库相关信息**
![](http://static.xiangdangnian.net.cn/20200907115641.png)

### 配置 nginx 

```bash
$ cd ~/laradock/nginx/sites
$ cp laravel.conf.example shop.conf
//修改配置文件如下图所示
$ vim shop.conf   
$ cd ~/laradock/
$ docker-compose restart nginx
```

![nginx 配置文件](http://static.xiangdangnian.net.cn/20200907120035.png)

### 更改项目所属用户

**由于权限问题，需要将项目的所属用户设置为 laradock 用户**

```
$ docker-compose exec workspace bash
# chown -R laradock:laradock laravel-shop
```

### 将前端项目打包

访问我们设置的域名后发现错误了。由于laravel 项目前端需要打包才能正常运行，下面执行打包操作。

![](http://static.xiangdangnian.net.cn/20200907142614.png) 

```bash
# npm install 
# npm run prod
```

### 重新访问项目

*如果能看到下面的内容就表示成功了*

![](http://static.xiangdangnian.net.cn/20200907142935.png)


![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)