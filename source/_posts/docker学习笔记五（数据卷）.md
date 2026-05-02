---
title: docker 学习笔记（五）
date: 2020-08-22 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/etZ4nrNVjob1F5phmvuzGifaOs7SWIkQ.jpg
tags:
  - docker
---

这一集下先从一张图开始
![数据管理](http://static.xiangdangnian.net.cn/7oOkIDb8rW10LGfRNhsvdp5MYcqltBXT.png)

这张图来自于 docker 官方，主要描述了主机和 docker 间的数据沟通的 3 种方式。分别是 **bind mount**、**volume**、**tmpfs mount**。这次主要学习前两种方式。让我们开始吧~

## volume (数据卷)
以下内容摘抄自 [docker 官方文档](https://docs.docker.com/storage) 

我们知道默认情况下，在容器内创建的所有文件都存储在可写容器层上，这意味着：
- 当容器不再存在时，数据不会持久存在，而且如果另一个进程需要数据，就很难从容器中取出数据。
- 容器可写层与容器运行的主机紧密耦合。您不能轻易地将数据移动到其他地方。
- 写入到容器的可写层需要一个存储驱动程序来管理文件系统。存储驱动程序提供使用Linux内核的联合文件系统。与使用直接写入主机文件系统的数据卷相比，这种额外的抽象降低了性能。

通过上面的内容告诉我们，**不要在容器内容写入数据,而要使用 volume**。

那么数据卷是怎么存储数据的呢？
> docker 默认在主机上会有一个特定的区域 /var/lib/docker/volumes/ (Linux)，该区域用来存放 volume。

### 实例演示

```bash
// 创建一个 volume 名为 my-data
$ docker volume create my-data
my-data

// 创建一个 名为 nginx1 的 nginx 容器并使用 my-data 为 nginx 的主目录
$ docker run -d -p 8080:80 --name nginx1  --mount source=nginx-data,target=/usr/share/nginx/html nginx

814dea34b7dbf2afd724d12ad50254e2a749f0b65684f04c730ac04181857d04

// 查看容器的状态
$ docker container ls --format "table {{.Image}}\t{{.ID}}\t{{.Status}}\t{{.Names}}"
IMAGE                   CONTAINER ID        STATUS              NAMES
nginx                   814dea34b7db        Up 3 minutes        nginx1

// 进入 nginx1 容器
$ docker exec -it nginx1 bash

// 修改 index.html 内容为 hello world
# echo hello world > /usr/share/nginx/html/index.html

// 访问 http://127.0.0.1:8080 我们可以看到 hello world 页面

// 删除 nginx1 容器
$ docker rm -f nginx1

// 创建一个 名为 nginx2 的 nginx 容器也使用 my-data 为 nginx 的主目录
$ docker run -d -p 8080:80 --name nginx2  --mount source=nginx-data,target=/usr/share/nginx/html nginx

// 访问 http://127.0.0.1:8080 我们也可以看到 hello world 页面（数据共享）
```

通过上面的实例我们知道了：
1. volume 和容器是分离的，删除容器并不会删除 volume
2. 多个容器可以加载相同的卷
3. volume 在任何系统的容器上都能工作，方便迁移

### 数据卷常用命令
- 查看所有数据卷  ```bash docker volume ls ```
- 创建数据卷 ```bash docker volume create [卷名称] ```
- 查看卷详情 ```bash docker volume inspect <卷名称> ```
- 删除数据卷 ```bash docker volume rm <卷名称> ```
- 删除无主的卷 ```bash docker volume prune ```

### 温馨提示：-v 和 --mount 的区别 (因为官方建议新用户使用 --mount，所以本文只记录 --mount 的使用方式)
> 在查看网上的各种资料的时候发现有人用 -v 参数而有的人又用 --mount 参数，为了确认他们之间的区别我特意查了一下 docker 官网，现在把内容放出来

- 最初，-v 或--volume 参数用于独立容器，而 --mount 参数用于群服务。 但是，从 Docker 17.06 开始，您也可以将 --mount 用于独立容器。 通常，--mount 更为明确和详细。 他们直接最大的区别是 -v 语法在一个字段中将所有选项组合在一起，而 --mount 语法将它们分开。 
- 新用户应该尝试—mount语法，它比—卷语法更简单。

---

**上面的内容主要讲述的是 local 驱动下的 volume，实际上 volume 还可以使用其他的驱动，这里先记录一下，后面在在继续学习。**

## bind mount 挂载主机目录

> 该方式实际上是将主机上的一个目录映射到容器中的一个目录。（使用过虚拟机的朋友应该比较熟悉）

**如果将空文件或目录挂载到容器，容器中的该目录又有文件，那么，这些文件将会被复制到主机上的目录中。如果将非空的文件或目录挂载到容器，容器中的该目录也有文件，那么，容器中的文件将会被隐藏。**

### 实例演示
```bash
// 在主机上创建一个 my-data 目录
$ mkdir ~/my-data

// 创建一个 名为 nginx3 的 nginx 容器并挂载 my-data 为 nginx 的主目录
$ cd ~
$ docker run -d -p 8080:80 --name nginx3 --mount type=bind,source="$(pwd)"/my-data,target=/usr/share/nginx/html nginx

// 在 my-data 目录中创建 index.html 并写入内容
$ touch my-data/index.html
$ echo hello bind type > my-data/index.html

// 使用浏览器访问 http://127.0.0.1:8080  可以正常显示 hello bind type

// 查看 nginx3 容器的详情，注意 mounts 部分，很清晰的显示了绑定的关系
$ docker inspect nginx3
......
"Mounts":[
    {
        "Type":"bind",
        "Source":"/Users/cc/my-data",
        "Target":"/usr/share/nginx/html"
    }
]
......

```

## tmpfs 方式 （以后学习）

> tmpfs，仅存储在主机系统的内存中，不会写入主机的文件系统。

----------

![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)