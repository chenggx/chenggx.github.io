---
title: docker 学习笔记（十）
date: 2020-09-6 22:22:22
author: chenggx
top: false
categories: Docker
img: http://static.xiangdangnian.net.cn/blog/2020/09/08/21-27-37-bac515a9f2b23f137a55b053ca6d883e-974f63.jpg

tags:
  - docker
---


> Swarm 是 Docker 引擎内置（原生）的集群管理和编排工具

学习 swarm 一定要理解的几个重要概念

- 节点
- 服务
- 任务

### 节点

一台物理或云主机加入 docker 集群，那么这台主机就是一个节点。

节点分为管理 (manager) 节点和工作 (worker) 节点。

管理节点用于集群的管理，一个 Swarm 集群可以有多个管理节点，但只有一个管理节点可以成为 leader。

工作节点是任务执行节点，管理节点将服务 (service) 下发至工作节点执行。

**管理节点默认也作为工作节点。你也可以通过配置让工作节点只进行任务调度。**


Docker 官网的这张图片形象的展示了集群中管理节点与工作节点的关系。

![管理节点与工作节点的关系](http://static.xiangdangnian.net.cn/20200831144229.png)

### 任务和服务

任务 （Task）是 Swarm 中的最小的调度单位，目前来说就是一个单一的容器。任务包含一个 Docker 容器和在容器中运行的命令。

服务 （Services） 是指一组任务的集合，服务定义了任务的属性。服务有两种模式：

- replicated services（副本模式） 按照一定规则在各个工作节点上运行指定个数的任务。
- global services（全局模式） 每个工作节点上运行一个任务

两种模式通过 docker service create 的 --mode 参数指定。


### swarm 集群基本操作

- 三台安装了 Docker 的主机，各主机间可以通信。
- Docker 版本必须大于 1.12
- 需要打开 tcp 2377 端口、tcp/udp 7946 端口、udp 4789 端口
- 选一台主机作为管理节点，获取其 ip 地址

#### docker-machine

Docker Machine 是 Docker 官方提供的一个工具，它可以帮助我们在远程的机器上安装 Docker，或者在虚拟机 host 上直接安装虚拟机并在虚拟机中安装 Docker。我们还可以通过 docker-machine 命令来管理这些虚拟机和 Docker。 

##### 安装 docker-machine

1. 首先下载可执行文件 https://github.com/docker/machine/releases
2. 将下载好的文件移动到 ```/usr/local/bin/docker-machine``` 目录下并改名为 docker-machine
3. 执行 ```sudo chmod +x /usr/local/bin/docker-machine ``` 命令为其添加可执行权限
4. 执行如下命令，验证是否安装成功
```bash
$ docker-machine -v
docker-machine version 0.16.1, build cce350d7
```

##### 初始化集群

第一步：如下命令创建 一个 Docker 主机作为管理节点。

```bash
$ docker-machine create -d virtualbox manager
```

由于众所周知的原因下载 boot2docker 特别慢，我们通过 [boot2docker 项目](https://github.com/boot2docker/boot2docker/releases/)页面直接下载该文件，然后将该文件移动到该 ```/Users/用户名/.docker/machine/cache``` 目录下。最后重新执行创建的命令就可以了


第二步：在管理节点上初始化一个 swarm 集群

```bash
// ssh 到虚拟机中
$ docker-machine ssh manager

// 初始化 swarm 集群，并指定管理节点 ip 地址
docker@manager:~$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

**执行 docker swarm init 命令的节点自动成为管理节点。**

第三步：创建工作节点，并加入 swarm 集群

worker 1 

```bash
$ docker-machine create -d virtualbox worker1

$ docker-machine ssh worker1

// 工作节点初始化集群的时候会自动给出该命令。
docker@worker1:~$ docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

This node joined a swarm as a worker.
```

**通过该 ```docker swarm join-token worker``` 命令获取加入集群的命令**

worker2

```bash
$ docker-machine create -d virtualbox worker2

$ docker-machine ssh worker2

docker@worker1:~$ docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

This node joined a swarm as a worker.
```

第四步：查看节点状态

```bash
// 该命令只能在管理节点使用
$ docker node ls

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
jnn6cji3hy22ninkymqxer49q *   manager             Ready               Active              Leader              19.03.12
r5sce06gn9351jvotf6xp1r53     worker1             Ready               Active                                  19.03.12
ssdfsdfs2df32dfk34dc3498      worker1             Ready               Active                                  19.03.12
```

**Docker 引擎通过主机名自动为节点命名**

**MANAGER 列标识群中的管理器节点。Leader 表示管理节点，空表示工作节点**

##### 部署一个服务

```bash
$ docker service create --replicas 1 --name helloworld busybox ping baidu.com

wo27l49vowb7reote7apzqn8d
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

- --preplicas 1 指定副本个数为 1 个（即在集群中启动一个容器运行）
- --name 指定服务名称为 helloworld
- busybox 指定镜像为 busybox
- ping baidu.com  指定运行的命令

##### 查看当前运行的服务

```bash
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                        PORTS
wo27l49vowb7        helloworld          replicated          1/1                 busybox:latesttcp
```

##### 查看服务详情

```bash
$ docker service inspect --pretty helloworld

ID:		wo27l49vowb7reote7apzqn8d
Name:		helloworld
Service Mode:	Replicated
 Replicas:	1
Placement:
UpdateConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Update order:      stop-first
RollbackConfig:
 Parallelism:	1
 On failure:	pause
 Monitoring Period: 5s
 Max failure ratio: 0
 Rollback order:    stop-first
ContainerSpec:
 Image:		busybox:latest@sha256:4f47c01fa91355af2865ac10fef5bf6ec9c7f42ad2321377c21e844427972977
 Args:		ping baidu.com
 Init:		false
Resources:
Endpoint Mode:	vip
```

- --pretty 以更好看的格式显示（不指定的话将以 json 格式显示）

##### 查看服务运行在哪个接点上

```bash
docker@manager:~$ docker service ps helloworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
re3rdqvnsoa3        helloworld.1        busybox:latest      worker1             Running             Running 11 minutes ago
```
我们可以看到该服务运行在 worker1 节点上。让我们 ssh 到 worker1 节点执行 docker ps 命令，发现确实有容器在运行。而我们 ssh 到其他节点上执行 docker ps 会发现并没有容器在运行。

##### 扩展集群中的服务

**必须在管理节点操作**

```bash
$ docker service scale helloworld=5

helloworld scaled to 5
overall progress: 5 out of 5 tasks
1/5: running   [==================================================>]
2/5: running   [==================================================>]
3/5: running   [==================================================>]
4/5: running   [==================================================>]
5/5: running   [==================================================>]
verify: Service converged

$ docker service ps helloworld

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE               ERROR                              PORTS
re3rdqvnsoa3        helloworld.1        busybox:latest      worker1             Running             Running about an hour ago
4tjmgtulmkx0        helloworld.2        busybox:latest      worker2             Running             Running 38 minutes ago
zl3u08rsrm72        helloworld.3        busybox:latest      manager             Running             Running 38 minutes ago
am36krqw6o97        helloworld.4        busybox:latest      manager             Running             Running 39 minutes ago
tsbhagfz6fx6        helloworld.5        busybox:latest      worker1             Running             Running 39 minutes ago
```

- 我们可以看到新增加了 4 个任务分布在manager、worker1、worker2 上。
- ssh 到各个主机执行 ```docker ps ``` 查看主机运行容器的状态

##### 删除集群中的服务

**必须在管理节点操作**

```bash
$ docker service rm helloworld
helloworld

docker@manager:~$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
```

**其他主机需要等一会再查看运行的容器是否删除**

##### 动态升级服务

创建一个低版本的 redis 服务

```bash
$ docker service create --replicas 3 --name redis --update-delay 10s redis:3.0.6

vep8ciixn4otmt9g0khd98z5d
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged
```

- --update-delay 设置更新任务之间的延迟时间（20h10m5s 表示延迟为 20 小时 10 分钟 5 表）
- --update-parallelism 设置同时更新任务的最大数（默认值为 1）
- --update-failure-action 设置更新任务失败后执行的动作(默认设置为如果在更新过程中的任何时候任务返回失败，则调度程序将暂停更新)

更新 redis

```bash
$ docker service update --image redis:3.0.7 redis
```

上面的命令实际执行流程为：

1. 暂停第一个任务
2. 为停止的任务安排更新计划
3. 启动一个新版本的容器
4. 如果任务的返回“RUNNING”，则等待指定的延迟时间后开始下一个任务
5. 如果任务返回“FAILED”，则暂停更新。


查看当前服务状态 

*可以看出服务的更新是暂停老版本，增加新版本*

```bash
docker@manager:~$ docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR                              PORTS
jbojkid2itrn        redis.1             redis:3.0.7         worker2             Running             Running 3 minutes ago
r4nyfe94e5ms         \_ redis.1         redis:3.0.6         worker2             Shutdown            Shutdown 3 minutes ago
t8umvvy4bf80        redis.2             redis:3.0.7         manager             Running             Running 2 minutes ago
rhunef6vi2dk         \_ redis.2         redis:3.0.6         manager             Shutdown            Shutdown 3 minutes ago
w2jcya0tc9cv        redis.3             redis:3.0.7         worker1             Running             Running 2 minutes ago
sz1ptkhmak6q         \_ redis.3         redis:3.0.6         worker1             Shutdown            Shutdown 2 minutes ago
```

##### 设置节点为 Drain 状态

目前我们的所有节点都为 Active 状态，也就是所有节点都可以接受任务。但有时候我们想指定某个节点正在维护，不进行任务处理，那么我们就要对节点的状态进行更改。

```bash
// 为了后面查看方便，删除上面进行创建的 redis 服务
$ docker service rm redis 

//重新创建一个 redis 服务
$ docker service create --replicas 3 --name redis --update-delay 10s redis:3.0.6

// 查看当前服务运行状态（每个节点运行一个任务）
$ docker service ps redis

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
ffhwf4i243yc        redis.1             redis:3.0.6         worker2             Running             Running 35 seconds ago
sflv38rtetvh        redis.2             redis:3.0.6         manager             Running             Running 35 seconds ago
kzn7cn15rrye        redis.3             redis:3.0.6         worker1             Running             Running 35 seconds ago

// 将 worker1 节点设为 Drain 状态
$ docker node update --availability drain worker1

worker1

// 查看当前服务运行状态（注意 worker1 节点服务的状态已经为关闭，worker2 节点上又多了一个任务）
$ docker service ps redis

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                     ERROR               PORTS
ffhwf4i243yc        redis.1             redis:3.0.6         worker2             Running             Running about a minute ago
sflv38rtetvh        redis.2             redis:3.0.6         manager             Running             Running about a minute ago
ir0nvesyhrea        redis.3             redis:3.0.6         worker2             Running             Running less than a second ago
kzn7cn15rrye         \_ redis.3         redis:3.0.6         worker1             Shutdown            Shutdown less than a second ago
```


### Routing Mesh （路由网格）

Routing Mesh 是 docker swarm 提供的确保服务在多节点网络上可用的集群网络机制。

当我们创建一个服务的时候，如果将端口发布出来，那么所有节点都会加入到这个路由网格中，当我们在外部通过任意主机 ip 和发布的端口访问服务的时候，无论该主机上时候启动服务，我们都能访问的到。（负载均衡）

在使用之前需要先开启如下端口：

- TCP/UDP 的 7946 端口
- UDP 的 4789 端口

格式： 

```bash
docker service create \
  --name <SERVICE-NAME> \
  --publish published=<PUBLISHED-PORT>,target=<CONTAINER-PORT> \
  <IMAGE>
```

- SERVICE-NAME 自定义服务的名称
- PUBLISHED-PORT 对外发布的端口号
- CONTAINER-PORT 容器的端口号
- IMAGE 服务所使用的镜像

#### 启动 nginx 服务并发布端口

```bash
$ docker service create --name web --publish published=8080,target=80 --replicas 2 nginx

// 查看服务当前运行在哪个节点上
$ docker service ps web

ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
kuocqhmt4rl7        web.1               nginx:latest        worker1             Running             Running 40 seconds ago
s3ybcckxdrnb        web.2               nginx:latest        manager             Running             Running 40 seconds ago

// 查看当前所有节点
$ docker node ls

ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
3px1nw94x2wmnr0k7846hpksh *   manager             Ready               Active              Leader              19.03.12
ibhxsuf055peywhawgp7hslav     worker1             Ready               Active                                  19.03.12
ny1e3wd4l7enyh0pp547azefe     worker2             Ready               Active                                  19.03.12
```

各节点 ip 地址
- manager 192.168.99.100
- worker1 192.168.99.101
- worker2 192.168.99.102


#### 访问各个节点

![访问结果](http://static.xiangdangnian.net.cn/blog/2020/08/31/21-35-55-45776d21fca64c07fad073412e998fa0-bdf130.png)

我们发现即使 worker2 节点没有运行 nginx 容器，但我们依然可以通过 worker2 节点访问发布的服务。

#### 对已存在的服务发布端口

```bash
docker service update \
  --publish-add published=<PUBLISHED-PORT>,target=<CONTAINER-PORT> \
  <SERVICE>
```

#### 绕过 Routing Mesh

不具体讲解了，没有想到实际的应用场景。


```bash
$ docker service create --name dns-cache \
  --publish published=53,target=53,protocol=udp,mode=host \
  --mode global \
  dns-cache
```
  
### 服务全局模式（global service）

上面的各种示例都是基于副本模式，下面开始全局模式。


```bash
$ docker service create \
  --mode global \
  --publish mode=host,target=80,published=8080 \
  --name=nginx \
  nginx:latest
```
  
使用上面的命令可以在当前的每个工作节点启动一个 nginx 容器，这时如果有一个节点挂掉了，那么并不会在其他节点重启启动一个新的容器。而如果我们这时在添加一个新的节点，那么当这个新的节点加入 swarm 集群的时候，该节点也会自动启动一个 nginx 容器。

### overlay 网络

> 该网络可以连接 swarm 集群中的一个或多个服务


#### 创建 overlay 网络

```bash
$ docker network create --driver overlay my-network
```

#### 创建服务的时候加入 overlay 网络

```bash
$ docker service create --replicas 3  --network my-network --name my-web nginx
```

#### 将现有的服务加入 overlay 网络

```bash
$ docker service update --network-add my-network my-web
```

#### 将现有服务从 overlay 网络中断开

```bash
$ docker service update --network-rm my-network my-web

```

### 在 Swarm 集群中管理敏感数据

#### 创建 secret 

格式：```docker secret create <secret-name> <file-name>```

```bash
$ printf "This is a secret" | docker secret create my_secret_data -
```
因为我们通过读取标准输入的方式，所以最后使用 - 标记

#### 创建一个 redis 服务并授权方位 secret 

```bash
$ docker service  create --name redis --secret my_secret_data redis:alpine
```

默认情况下，容器可以通过 ```/run/secrets/<secret_name>``` 路径访问密钥。或者可以使用 target 选项自定义容器上的文件名。

#### 查看当前服务运行状态

```
$ docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
bp0ylcmxw6ll        redis.1             redis:alpine        worker1             Running             Running about a minute ago
```

#### 验证 secret 

```bash
// 根据上面查看的数据，进入 worker1 容器
$ docker-machine ssh worker1

// 获取 worker1 容器运行的 redis 服务
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
aa049d4173e1        redis:alpine        "docker-entrypoint.s…"   8 minutes ago       Up 8 minutes        6379/tcp            redis.1.bp0ylcmxw6llv9icvhgrk189f

// 进入 redis 容器
$ docker exec -it redis.1.bp0ylcmxw6llv9icvhgrk189f sh

//在容器中查看 secret
# cat /run/secrets/my_secret_data
This is a secret
```

#### 使用中的 secret 无法删除

```bash
// 进入 manager 节点
$ docker-machine ssh manager

$ docker secret rm my_secret_data
Error response from daemon: rpc error: code = InvalidArgument desc = secret 'my_secret_data' is in use by the following service: redis
docker@manager:~$
```

#### 通过更新的方式删除 secret

```bash
$ docker service update --secret-rm my_secret_data redis
redis
overall progress: 1 out of 1 tasks
1/1: running   [==================================================>]
verify: Service converged
```

#### 删除服务和 secret 

```bash
$ docker service rm redis

$ docker secret rm my_secret_data
```


### 在 Swarm 集群中管理非敏感数据（配置）

**使用方式与 secret 相似**

> docker 17.06引入了swarm service configs。允许你在服务映像或运行容器之外存储非敏感信息

**注意：config 仅能在 Swarm 集群中使用。**

#### 创建 config

```bash
// 创建一个名为 redis.conf 的文件，内容为 port：6380
$ cat redis.conf
port 6380

// 创建 config
$ docker config create redis.conf redis.conf

$ docker config ls
ID                          NAME                CREATED             UPDATED
fkamdbup3egw8752bm4ixrsm2   redis.conf          23 minutes ago      23 minutes ago
```

跟 secret 使用类似


#### 创建 redis 服务并授权其方位 config 

```bash
$ docker service create \
--name redis \
--config redis.conf \
--publish published=6379,target=6380 \
redis:latest \
redis-server /redis.conf
```

或显式的指定路径

```bash
$ docker service create \
--name redis \
--config source=redis.conf,target=/etc/redis.conf \
--publish published=6379,target=6380 \
redis:latest \
redis-server /etc/redis.conf
```

以上两种创建服务的方法都可以

#### 验证 config 

使用 redis 管理工具连接 redis，链接地址填写 192.168.99.100

![连接配置](http://static.xiangdangnian.net.cn/20200903172350.png)
![连接结果](http://static.xiangdangnian.net.cn/20200903172402.png)


#### 使用中的 config 无法删除

```bash
$ docker config rm redis.conf
Error response from daemon: rpc error: code = InvalidArgument desc = config 'redis.conf' is in use by the following service: redis
```

#### 通过更新的方式删除 config

```bash
docker service update --config-rm redis.conf redis

docker config rm redis.conf
redis.conf
```

#### 移除 redis 服务

```bash
$ docker service rm redis
```

![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)