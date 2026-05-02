---
title: docker 学习笔记（八）
date: 2020-08-27 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/20200828083314.jpg
tags:
  - docker
---

# 使用 docker-compose 搭建 LNMP 开发环境

上一集我们已经可以通过 docker 搭建 LNMP 的开发环境了，但是想必大家也发现配置挺复杂的，每个容器启动都有好长的命令。那有没有更简单一点的方式呢？有的，就是今天要学习的——docker-compose。

> 什么是 docker-compose 呢？

docker-compose 是一个使用 python 编写，用于定义和运行多容器的工具。

## 安装

### 二进制包安装

```bash
// 由于网络原因可以将文件直接下载下来，然后放到对应的位置，最后赋予相应的执行权限也是一样的
$ sudo curl -L https://github.com/docker/compose/releases/download/1.25.5/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

$ sudo chmod +x /usr/local/bin/docker-compose
```


### pip 安装

```bash
$ sudo pip install -U docker-compose
```

#### bash 补全命令

使用如下命令使 docker-compose 具有代码提示功能（如不生效，可以退出终端重新进入就可以了）

```bash
$ curl -L https://raw.githubusercontent.com/docker/compose/1.25.5/contrib/completion/bash/docker-compose > /etc/bash_completion.d/docker-compose
```

## 卸载

通过二进制包安装的卸载方式

```bash
$ sudo rm /usr/local/bin/docker-compose
```

通过 pip 安装的卸载方式

```bash
$ sudo pip uninstall docker-compose
```

## 通过 docker-compose 配置 LNMP 开发环境

> 我们直接将上一集中配置的 LNMP 环境通过docker-compose 的方式在配置一遍

主要步骤如下：

1. 创建一个目录 easy-docker 作为 docker-compose 目录
2. 运行一个临时的 nginx 容器，将 nginx 配置文件复制到 easy-docker 目录中，并修改配置文件
3. 在 easy-docker 目录中创建 php 目录并在该目录下创建 Dockerfile 文件
4. 编辑 phpfpm 目录下的 Dockerfile 文件
5. 在 easy-docker 目录下创建 docker-compose.yml 文件并编辑
6. 使用 docker-compose up -d 运行

```bash
// 创建一个目录用于保存 docker-compose 项目所需的内容
$ mkdir easy-docker && cd easy-docker

// 运行一个临时的 nginx 容器并将配置文件复制到当前目录
$ docker run --name temp-nginx -d nginx

$ docker cp temp-nginx:/etc/nginx ./

// 删除临时 nginx 容器
$ docker rm -f temp-nginx

// 修改 nginx 配置文件（详情见下图）
$ vim nginx/conf.d/default.conf

// 创建一个目录作为 nginx 容器项目主目录
$ mkdir wwwroot

// 创建 phpfpm 目录
$ mkdir phpfpm && cd phpfpm

// 创建 Dockerfile 文件（见下图）
$ vim Dockerfile

// 在 easy-docker 目录下创建 docker-compose.yml 文件（见下文）
$ cd ..
$ vim docker-compose.yml

//构建镜像并启动容器
$ docker-compose up

// 测试
$ echo "<?php phpinfo(); ?>" > ./wwwroot/info.php

// 访问页面可以看到 phpinfo 页面怎成功
```

---

nginx 配置文件

![nginx 配置文件](http://static.xiangdangnian.net.cn/20200826104323.png)

---

phpfpm Dockerfile 内容

```bash
FROM php:7.2-fpm
RUN sed -i "s@http://deb.debian.org@http://mirrors.aliyun.com@g" /etc/apt/sources.list \
      && rm -Rf /var/lib/apt/lists/* \
      && apt-get update && apt-get install -y \
                libfreetype6-dev \
                libjpeg62-turbo-dev \
                libpng-dev \
        && docker-php-ext-install -j$(nproc) iconv \
        && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
        && docker-php-ext-install -j$(nproc) gd \
   && docker-php-ext-install mysqli pdo pdo_mysql
```

---

docker-compose 文件内容

![](http://static.xiangdangnian.net.cn/20200827171943.png)


上面的 docker-compose.yml 文件只展示了部分指令的用法，可以参考[链接](https://yeasy.gitbook.io/docker_practice/compose/compose_file)查看其他指令的详细用法。


## docker-compose 常用命令

该命令非常常用，主要功能是尝试自动完成包括构建镜像，（重新）创建服务，启动服务，并关联服务相关容器的一系列操作。

```bash
$ docker-compose up -d      
```

将会停止 up 命令所启动的容器，并移除网络

```bash
$ docker-compose down
```

其他一些命令类似有 docker 相关命令，只是把关键指令换成 docker-compose 了，例如 docker-compose ps、docker-compose exec 等，可自行尝试。 [相关文档](https://yeasy.gitbook.io/docker_practice/compose/commands#exec)


## 优秀项目参考

上面我们只是简单的搭建了一个开发环境，实际上真实的环境会很复杂，我们自己写一个完整的 docker-compose 文件可能比较困难（大神忽略）。那么有什么简单的方法呢？

下面我推荐两个我用过还不错的项目，大家可以参考学习。

[laradock](https://github.com/laradock/laradock)

[LNMP](https://github.com/khs1994-docker/lnmp)



![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)