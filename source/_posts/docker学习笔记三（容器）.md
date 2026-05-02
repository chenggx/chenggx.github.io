---
title: docker 学习笔记（三）
date: 2020-08-17 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/pexels-karol-d-409701.jpg
tags:
  - docker
---

## docker 容器

容器是通过 image 创建的进程。
> 镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。

### 新建并启动容器
下面的命令的含义：通过 Ubuntu:18.04 这个 image 创建一个容器并运行 /bin/echo 'Hello world'，完成后停止该容器。
```bash
$ docker run ubuntu:18.04 /bin/echo 'Hello world'
Hello world
```

使用 docker run 命令，后台实际上执行的内容为：

1. 检查本地是否存在指定的镜像，不存在就从公有仓库下载
2. 利用镜像创建并启动一个容器
3. 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层（后面会学）
4. 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去（后面会学）
5. 从地址池配置一个 ip 地址给容器（后面会学）
6. 执行用户指定的应用程序
7. 执行完毕后容器被终止

上面的示例执行完以后会终止，但是一般我们在使用一个提供服务的容器的时候，不想让它停止，那么可以使用 -d 参数，使容器保持在后台运行。但是需要注意——**容器是否会长久运行，是和 docker run 指定的命令有关，和 -d 参数无关**

如下命令执行完后，容器依然会停止
```bash
$ docker run -d ubuntu
```
![已停止](http://static.xiangdangnian.net.cn/XF6zB8vbNWw9uPLGrSkiIt3YdZaHc0K7.png)

而如下命令执行完后，容器则在后台保持运行
```bash
$ docker run -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```
![运行中](http://static.xiangdangnian.net.cn/VcoLwKIYlG65etAO1bCWNfHUES9FpBza.png)

### 进入容器

#### attach 命令

```bash
$ docker run -dit ubuntu
ffff9516c6151ef3b436df1bccc70ba9da2d0f57bbec5afe19353fe481e12702

$ docker container ls
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                                      NAMES
ffff9516c615        ubuntu                  "/bin/bash"              7 seconds ago       Up 6 seconds                                                   elegant_hypatia

$ docker attach ffff9516c615
root@ffff9516c615:/#
```
**注意： 如果从这个 stdin 中 exit，会导致容器的停止。**

#### exec 命令(推荐使用，一般配合 -it 参数)

```bash
$ docker run -dit ubuntu
15fc4d97c1b4ea25d76e568fb4e695d5b48d7f13ebbb6d718a80b86a4764a005

// 只用 -i 参数时，由于没有分配伪终端，界面没有我们熟悉的 Linux 命令提示符，但命令执行结果仍然可以返回。
$ docker exec -i 15fc4d97c1b4ea25 bash
ls
bin
boot
dev
etc
...

//当 -i -t 参数一起使用时，则可以看到我们熟悉的 Linux 命令提示符。
$ docker exec -it 15fc4d97c1b4ea25 bash
root@15fc4d97c1b4:/#
```
**注意：如果从这个 stdin 中 exit，不会导致容器的停止。**

#### 容器常用命令
1. docker container ls --all        **查看当前系统中的所有（运行中、已停止的）容器** 
2. docker container start XXX       **把已经停止的 XXX 容器启动**
3. docker container stop XXX        **把运行总的 XXX 容器停止** 
4. docker container restart XXX     **重新启动运行中的 XXX 容器**
5. docker container prune           **删除所有处于停止状态的容器**


![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)