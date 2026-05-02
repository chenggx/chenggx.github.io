---
title: docker 学习笔记（二）
date: 2020-08-10 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/blog/2020/08/10/21-07-53-16d1a6fa8852ba7541126a8803fb37bb-75f883.jpg
tags:
  - docker
---

# 镜像 Image

## 简介
> Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

上面是比较官方的解释，估计大部分也没看太懂，那么我就用我自己理解的方式说一下吧。

image 类似于我们安装系统的镜像文件，通过 image 文件我们可以生成容器文件。一般镜像文件是分层存储的，使用了Union Fs 的技术（具体是个啥我也不太懂🤦‍♂️），也就说一个镜像文件是很多块组成的，有点类似于现在前端的组件化开发，是一组文件组成的。镜像可以向面向对象的类一样可以进行继承，通过一些基础镜像来构建属于我们自己的镜像。


上面说了一大堆好像还不是很明白的样子，下面还是用例子来说明吧。go go go~~~

## 获取镜像
> 通过 docker pull 命令从 Docker Hub 仓库获取镜像。

```bash
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```
具体的选项可以通过 docker pull --help 查看

1. 地址的格式一般是 <域名/IP>[:端口号] 默认地址是 Docker Hub, 可以省略
2. 仓库名：这里的仓库名是两段式名称，即 <用户名>/<软件名>。对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。
3. 标签： 一般为版本号

**这里穿插一个镜像急速的内容**
由于国内网络的问题我们需要对拉取镜像的地址镜像更改，使用国内的镜像地址来加快拉取速度。

### linux 系统（已测试）
在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）
```json
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```
重启服务
```bash
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

### Windows 10（未测试）
在任务栏托盘 Docker 图标内右键菜单选择 Settings，打开配置窗口后在左侧导航菜单选择 Docker Engine，在右侧像下边一样编辑 json 文件，之后点击 Apply & Restart 保存后 Docker 就会重启并应用配置的镜像地址了。
```json
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

### macOS。（已测试）
在任务栏点击 Docker Desktop 应用图标 -> Perferences，在左侧导航菜单选择 Docker Engine，在右侧像下边一样编辑 json 文件。修改完成之后，点击 Apply & Restart 按钮，Docker 就会重启并应用配置的镜像地址了。
```json
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

执行 **$ docker info**如果从结果中看到如下内容，说明配置成功。
```bash
...
Registry Mirrors:
 https://hub-mirror.c.163.com/
...
```


## 继续镜像的拉取
使用 docker pull 拉取Ubuntu 镜像

```bash
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
7595c8c21622: Pull complete
d13af8ca898f: Pull complete
70799171ddba: Pull complete
b6c12202c5ef: Pull complete
Digest: sha256:a61728f6128fb4a7a20efaa7597607ed6e69973ee9b9123e3b4fd28b7bba100b
Status: Downloaded newer image for ubuntu:18.04
docker.io/library/ubuntu:18.04
```
上面的命令中没有给出 Docker 镜像仓库地址，因此将会从 Docker Hub 获取镜像。而镜像名称是 ubuntu:18.04，因此将会获取官方镜像 library/ubuntu 仓库中标签为 18.04 的镜像。

从下载过程中可以看到我们之前提及的分层存储的概念，镜像是由多层存储所构成。下载也是一层层的去下载，并非单一文件。

## 以镜像为基础启动容器
```bash
$ docker run -i -t --rm ubuntu:18.04 bash
root@94053f3fa153:/# cat /etc/os-release
NAME="Ubuntu"
VERSION="18.04.4 LTS (Bionic Beaver)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 18.04.4 LTS"
VERSION_ID="18.04"
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
VERSION_CODENAME=bionic
UBUNTU_CODENAME=bionic
root@94053f3fa153:/#
```
docker run 就是运行容器的命令，具体格式在后面容器相关章节进行详细介绍。这里简单解释一下
- -i 交互式操作，让容器的标准输入保持打开。
- -t 选项让 Docker 分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上。
- --rm 容器退出后随之将其删除。（默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动 docker rm。我们这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用 --rm 可以避免浪费空间。）
- ubuntu：18.04  使用 ubuntu:18.04 镜像为基础来启动容器。
- bash 放在镜像名后的是 命令，这里我们希望有个交互式 Shell，因此用的是 bash。

进入容器后，我们可以在 Shell 下操作，执行任何所需的命令。这里，我们执行了 cat /etc/os-release，这是 Linux 常用的查看当前系统版本的命令，从返回的结果可以看到容器内是 Ubuntu 18.04.1 LTS 系统。

最后我们通过 exit 退出了这个容器。

## 列出镜像
使用 docker image ls 列出已经下载了的镜像文件
```bash
$ docker image ls
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
redis                latest              5f515359c7f8        5 days ago          183 MB
nginx                latest              05a60462f8ba        5 days ago          181 MB
mongo                3.2                 fe9198c04d62        5 days ago          342 MB
<none>               <none>              00285df0df87        5 days ago          342 MB
ubuntu               18.04               f753707788c5        4 weeks ago         127 MB
ubuntu               latest              f753707788c5        4 weeks ago         127 MB
```
列表包含了 仓库名、标签、镜像 ID、创建时间 以及 所占用的空间。

- 镜像 ID 则是镜像的唯一标识，因为一个镜像可以对应多个标签。因此，在上面的例子中，我们可以看到 ubuntu:18.04 和 ubuntu:latest 拥有相同的 ID，因为它们对应的是同一个镜像。

- 镜像的体积跟 Docker Hub 上的不一致是因为 Docker Hub 中显示的体积是压缩后的体积。而 docker image ls 显示的是镜像下载到本地后，展开的大小，准确说，是展开后的各层所占空间的总和。

- 另外一个需要注意的问题是，docker image ls 列表中的镜像体积总和并非是所有镜像实际硬盘消耗。由于 Docker 镜像是多层存储结构，并且可以继承、复用，因此不同镜像可能会因为使用相同的基础镜像，从而拥有共同的层。由于 Docker 使用 Union FS，相同的层只需要保存一份即可，因此实际镜像硬盘占用空间很可能要比这个列表镜像大小的总和要小的多。

使用以下命令查看镜像、容器、数据卷所占用的空间。
```bash
$ docker system df

TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              24                  0                   1.992GB             1.992GB (100%)
Containers          1                   0                   62.82MB             62.82MB (100%)
Local Volumes       9                   0                   652.2MB             652.2MB (100%)
Build Cache                                                 0B                  0B
```

### 虚悬镜像
```bash
<none>               <none>              00285df0df87        5 days ago          342 MB
```
既没有仓库名，也没有标签,显示为 none 的就是虚悬镜像

个镜像原本是有镜像名和标签的，原来为 mongo:3.2，随着官方镜像维护，发布了新版本后，重新 docker pull mongo:3.2 时，mongo:3.2 这个镜像名被转移到了新下载的镜像身上，而旧的镜像上的这个名称则被取消，从而成为了 <none>。除了 docker pull 可能导致这种情况，docker build 也同样可以导致这种现象。由于新旧镜像同名，旧镜像名称被取消，从而出现仓库名、标签均为 <none> 的镜像。这类无标签镜像也被称为 虚悬镜像(dangling image) ，可以用下面的命令专门显示这类镜像：

```bash
$ docker image ls -f dangling=true
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              00285df0df87        5 days ago          342 MB
```

一般来说，虚悬镜像已经失去了存在的价值，是可以随意删除的，可以用下面的命令删除。
```bash
$ docker image prune
```

### 中间层镜像
为了加速镜像构建、重复利用资源，Docker 会利用 中间层镜像。所以在使用一段时间后，可能会看到一些依赖的中间层镜像。默认的 docker image ls 列表中只会显示顶层镜像，如果希望显示包括中间层镜像在内的所有镜像的话，需要加 -a 参数。


```bash
$ docker image ls -a
```

**这些无标签镜像不应该删除，否则会导致上层镜像因为依赖丢失而出错**

### 常用列出镜像的命令
#### 根据仓库名列出镜像

```bash
$ docker image ls ubuntu
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              18.04               f753707788c5        4 weeks ago         127 MB
ubuntu              latest              f753707788c5        4 weeks ago         127 MB
```

#### 列出特定的某个镜像

```bash
$ docker image ls ubuntu:18.04
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              18.04               f753707788c5        4 weeks ago         127 MB
```

#### 只查看镜像ID
```bash
$ docker image ls -q
5f515359c7f8
05a60462f8ba
fe9198c04d62
00285df0df87
f753707788c5
f753707788c5
1e0c3dd64ccd
```

#### 以表格等距显示，并且有标题行
```bash
$ docker image ls --format "table {{.ID}}\t{{.Repository}}\t{{.Tag}}"
IMAGE ID            REPOSITORY          TAG
5f515359c7f8        redis               latest
05a60462f8ba        nginx               latest
fe9198c04d62        mongo               3.2
00285df0df87        <none>              <none>
f753707788c5        ubuntu              18.04
f753707788c5        ubuntu              latest
```

## 删除镜像
> 格式
```bash
$ docker image rm [选项] <镜像1> [<镜像2> ...]
```
**其中，<镜像> 可以是 镜像短 ID、镜像长 ID、镜像名 或者 镜像摘要。**

### Untagged 和 Deleted
因为一个镜像可以有多个标签，当我们执行 ```docker image rm``` 命令时，如果如果还有其他标签指向这个镜像，那么就不会产生 Delete 操作。

### 用 docker image ls 命令来配合删除

以下命令可以删除所有仓库名为 redis 的镜像
```bash
$ docker image rm $(docker image ls -q redis)
```

![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)