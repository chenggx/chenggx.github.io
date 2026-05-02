---
title: docker 学习笔记（一）
date: 2020-08-09 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/blog/2020/08/10/20-46-42-247cb7b419a0fbdc33d90c75ceea78cb-cfef34.jpg
tags:
  - docker
---

## docker 介绍
docker 这个东西应该做开发的应该都有听说过，但是不知道大家有没有详细的了解过。反正我是属于听说过，大概知道是个什么东西，能特别简单的使用。但是一直没有深入的学习过，今天在这里我想把学习 docker 的过程记录下来，方便日后查找。

那么下面开始啦。~~~

## 为啥要使用 docker
> 在我电脑上运行的是正常的啊？怎么到了别人的电脑上就不能正常运行了呢？

如果你也碰到上面的问题了，那么请你也开始 docker 的学习吧。这个时候有人会说为啥不用虚拟机呢，因为虚拟机是完全模拟一台正常运行的。就算你只是需要运行一个 php 的环境，那么系统默认的一些其他进程也是会启动的，这样就会降低我们电脑的利用率。还有一点就是虚拟机一般是不能再生产环境中使用的。但是 docker 没有问题。总而言之一句话，虚拟机不好，我们要用 docker 。

## docker 的用途
1. 提供一次性的环境。比如，本地测试他人的软件、持续集成的时候提供单元测试和构建的环境。
2. 提供弹性的云服务。因为 Docker 容器可以随开随关，很适合动态扩容和缩容。
3. 组建微服务架构。通过多个容器，一台机器可以跑多个服务，因此在本机就可以模拟出微服务架构。

## docker 的安装
docker 分为社区版（CE）、企业版本（EE）
win 和 mac 安装很简单，就是正常的安装软件的方法，而用能用 linux 系统的，相信你也有办法找到安装的方法的。

安装完成之后使用如下命令验证是否安装成功

```bash
$ docker version
# 或者
$ docker info
```
**Docker 需要用户具有 sudo 权限，为了避免每次命令都输入sudo，可以把用户加入 Docker 用户组**

```bash
$ sudo usermod -aG docker $USER
```
使用如下命令启动 docker
```bash
# service 命令的用法
$ sudo service docker start

# systemctl 命令的用法
$ sudo systemctl start docker
```

## 第一个实例 hello world
```bash
$ docker container run hello-world
```
如果运行成功，你会在屏幕上读到下面的输出。
```bash
$ docker container run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

... ...
```

上面命令的过程就是：
1. 从 docker 的官方仓库拉取 hello world image 文件到本地
2. 使用 image 文件 生成 容器文件
3. 运行 hello world 容器 ---> 输出相关内容  ---> 容器自动终止（有些容器不会自动终止，因为提供的是服务。比如，安装运行 Ubuntu 的 image，就可以在命令行体验 Ubuntu 系统。）


通过上面的实例大概知道了 docker 的是怎么回事了。主要就是需要明白两个文件，image文件和容器文件。接下来我们制作一个 image 文件，并通过该文件生成容器并运行。

## 制作自己的 docker 容器
1. 新建一个目录
```bash
$ mkdir hello && cd hello 
```
2. 在目录中新建 .dockerignore 文件（该文件类似于 git 项目中的 .gitignore 文件）
3. 在目录中新建 Dockerfile 文件,内容如下所示
![](http://static.xiangdangnian.net.cn/blog/2020/08/10/20-29-32-9faae2396ce4759958aee55bfcabc924-f01fa1.png)

4. 运行命令，进行 image 的构建
```bash
$ docker build -t nginx:v1 .
```
看到如下内容则表示 image 构建成功。
![](http://static.xiangdangnian.net.cn/blog/2020/08/10/20-30-47-a92e94e81727959af077be139e676ed6-044d5a.png)

5. 通过 image 文件生成容器文件
```bash
$ docker run --name webserver -p 9527:80 nginx:v1
```
6. 通过浏览器访问 localhost:9527,如果可以看到如下内容则成功
![](http://static.xiangdangnian.net.cn/blog/2020/08/10/20-35-50-6f66e88683370b594f2ec01744c008cf-0c3367.png)

一个简单的容器就完成了。通过这个例子我们可以大概了解 docker 的配置，运行， 后面会深入 docker 相关的概念进行学习，并定下第一个计划 ———— 配置一个属于自己的 LNMP docker 测试环境。

![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)