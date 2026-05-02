---
title: docker 学习笔记（六）
date: 2020-08-24 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/20200826165158.jpg
tags:
  - docker
---

# 使用 Dockerfile 定制镜像

> 什么是 Dockerfile 呢？

Dockerfile 是一个文本文档，其中包含用户可以在命令行上调用以组装映像的所有命令。Docker 可以通过阅读该文件中的指令来自动构建映像。（类似于 Linux 上的 bash 脚本，Docker 通过该脚本构建镜像）

## 使用 dockerfile 制作一个 nginx 镜像

```bash
$ mkdir mynginx
$ cd mynginx
$ touch Dockerfile   //首字母必须大写
```

Dockerfile 文件内容如下

```
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

这个 Dockerfile 很简单，一共就两行。涉及到了两条指令，FROM 和 RUN。

### FROM
功能：指定基础镜像

> 所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。就像我们之前运行了一个 nginx 镜像的容器，再进行修改一样，基础镜像是必须指定的。而 FROM 就是指定 基础镜像，因此一个 Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令。

一般使用中我们通过 [Docker Hub](https://hub.docker.com/search?q=&type=image&image_filter=official) 来查找相关镜像。如下图所示，红标中标识的为官方镜像
![docker hub](http://static.xiangdangnian.net.cn/20200826144635.png)

除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch（该镜像不能通过 docker pull 命令直接拉取）。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

*因为本人只对 PHP 较为熟悉，没有使用过 go，这个也不是很了解，就先跳过了*

### RUN
功能：执行命令

> 用来执行命令行命令的

实际使用下有两种格式

1. shell 格式： ```RUN <命令>```
2. exec 格式：```RUN ["可执行文件", "参数1", "参数2"]```

Dockerfile 中每一个指令都会建立一层，RUN 也不例外。每一个 RUN 执行结束后，都会 commit 这一层的修改，构成新的镜像。所以在使用中尽力减少指令。

```
FROM debian:stretch

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make wget
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

上面的这种写法，创建了 7 层镜像。这是完全没有意义的，而且很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。我们在以后的使用应该避免。正确的写法如下所示：

```
FROM debian:stretch

RUN buildDeps='gcc libc6-dev make wget' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-5.0.3.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

在上一个 Dockerfile 中所有的命令只有一个目的，就是编译、安装 redis 可执行文件。因此没有必要建立很多层，这只是一层的事情。因此，这里我们仅仅使用一个 RUN 指令，并使用 && 将各个所需命令串联起来。将之前的 7 层，简化为了 1 层。

**在撰写 Dockerfile 的时候，要经常提醒自己，这并不是在写 Shell 脚本，而是在定义每一层该如何构建。**

## 构建

命里格式： ```docker build [选项] <上下文路径/URL/->```

```bash
$ docker build -t mynginx:v1 .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 08393e824c32
Step 2/2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in 1d7edd724b5f
Removing intermediate container 1d7edd724b5f
 ---> e29ba82c8e43
Successfully built e29ba82c8e43
Successfully tagged mynginx:v1

// 查看刚刚创建的镜像
$ docker image ls
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
mynginx                 v1                  e29ba82c8e43        3 minutes ago       132MB
```

1. -t 表示指定镜像的名称和标签，格式为 name:tag
2. 最后有个点，表示构建的上下文路径为当前目录

## 其他构建方式

1. 通过 Git repo 进行构建
2. 使用 tar 包进行构建


## 常用指令介绍

#### COPY 复制文件
格式：

- ```COPY [--chown=<user>:<group>] <源路径> <目标路径> ```  (常用)
- ```COPY [--chown=<user>:<group>] ["<源路径1>", "<目标路径>"] ```

构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置。

```bash
COPY package.json /usr/src/app/
```

加上 --chown=<user>:<group> 选项来改变文件的所属用户及所属组

```bash
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
COPY --chown=1 files* /mydir/
COPY --chown=10:11 files* /mydir/
```

#### ADD 更高级的复制

和 COPY 的格式和性质基本一致。但是在 COPY 基础上增加了一些功能。

- <源路径> 可以是一个 URL。Docker 引擎会试图去下载这个链接的文件放到 <目标路径> 去。下载后的文件权限自动设置为 600
- <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。

**在 Docker 官方的 Dockerfile 最佳实践文档 中要求，尽可能的使用 COPY，因为 COPY 的语义很明确，就是复制文件而已，而 ADD 则包含了更复杂的功能，其行为也不一定很清晰。最适合使用 ADD 的场合，就是所提及的需要自动解压缩的场合。**

#### CMD 容器启动命令

> Docker 不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。CMD 指令就是用于指定默认的容器主进程的启动命令的。

格式：

- shell 格式：CMD <命令>
- exec 格式：CMD ["可执行文件", "参数1", "参数2"...]   （推荐，一定要使用双引号）

#### ENV 设置环境变量

> 用于设置环境变量，无论是后面的其它指令，如 RUN，还是运行时的应用，都可以直接使用这里定义的环境变量。

格式：

- ENV ```<key> <value>```
- ENV ```<key1>=<value1> <key2>=<value2>...```

#### ARG 构建参数

格式：

- ```ARG <参数名>[=<默认值>]```

> 与 ENV 指令一样，都是设置环境变量。所不同的是，ARG 所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。

Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 ```--build-arg <参数名>=<值>``` 来覆盖。

#### VOLUME 定义匿名卷

格式：

- ```VOLUME ["<路径1>", "<路径2>"...]```
- ```VOLUME <路径>```

示例：

容器中的 /data 目录自动挂载到匿名卷中

```bash
VOLUME /data
```

**该指令可以在运行时被覆盖**

```bash
docker run -d -v mydata:/data xxxx
```

#### EXPOSE 暴露端口

格式：

- ```EXPOSE <端口1> [<端口2>...] ```

声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明就会开启这个端口的服务。

**写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 docker run -P 时，会自动随机映射 EXPOSE 的端口。**

#### WORKDIR 指定工作目录

格式：

- WORKDIR <工作目录路径>。

使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录。

```bash
RUN cd /app
RUN echo "hello" > world.txt
```

使用上面的内容构建镜像后会发现根本找不到 ```/app/world.txt``` 文件。原因其实很简单，在 Shell 中，连续两行是同一个进程执行环境，因此前一个命令修改的内存状态，会直接影响后一个命令；而在 Dockerfile 中，这两行 RUN 命令的执行环境根本不同，是两个完全不同的容器。这就是对 Dockerfile 构建分层存储的概念不了解所导致的错误。

**因此如果需要改变以后各层的工作目录的位置，那么应该使用 WORKDIR 指令。**

#### USER 指定当前用户

格式：

- ```USER <用户名>[:<用户组>]```

USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。WORKDIR 是改变工作目录，USER 则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。当然，和 WORKDIR 一样，USER 只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。

*其他指令参考官方文档*


![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)