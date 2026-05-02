---
layout: php
title: laravel 使用 sail 开发技巧
date: 2024-02-29 20:24:36
author: chenggx
top: false
tags: 
  - php
  - sail
---
  
  ## 前置条件

1. 首先需要需要确保当前系统已经安装好了 docker 环境。
2. 当前系统的 php 版本于所需 laravel 的依赖版本不一致。

## 步骤

1. 安装 laravel 项目

> 在系统中使用如下命令安装 laravel 项目 _my-laravel-app_，其中的 _php83_ 可更换为所需的 php 版本，如 74、80、81。

```bash
# docker run --rm -v $(pwd):/var/www -w /var/www laravelsail/php83-composer:latest composer create-project --prefer-dist laravel/laravel my-laravel-app
```

2. 安装所需的 sail 服务
   1. 进入容器
   2. 进入项目目录
   3. 使用命令添加 sail 服务
   4. 使用空格键选择所需的服务后，按回车键后安装，项目目录中会生成 _docker-compose.yml_ 文件

```bash
# docker run -it --rm -v $(pwd):/var/www -w /var/www laravelsail/php83-composer:latest bash
# cd my-laravel-app
# php artisan sail:add
```

3. 查看 _docker-compose.yml_ 文件内容并根据需要进行修改（一般不需要修改）

4. 修改 _.env_ 文件

   > 因系统端口号可能有冲突，可以在 _.env_ 根据 _docker-compose.yml_ 中的变量进行添加，以下是常用的几个变量需要修改，其他的根据具体需求进行修改。

   - APP_PORT 为项目的端口号
   - FORWARD_DB_PORT 为数据库的端口号
   - FORWARD_REDIS_PORT 为 redis 的端口号

5. 运行项目
   > 使用下面的命令以后台的方式运行项目

```bash
# ./vendor/bin/sail up -d
```

6. 查看项目是否正常
   1. 使用浏览器打开 127.0.0.1:APP_PORT 查看页面是否正常
   2. 使用数据库工具连接 mysql
   3. 使用数据库工具连接 redis

## 其他是用技巧

### 配置 sail 别名访问

将以下内容添加到 ~/.zshrc 或 ~/.bashrc ，然后重新启动你的 shell，就可以是用 sail 来执行命令了。

```bash
alias sail='[ -f sail ] && sh sail || sh vendor/bin/sail'
```

### 执行 php 命令

```bash
# sail php --version

# sail php script.php
```

### 执行 Composer 命令

```bash
# sail composer require laravel/sanctum
```

### 执行 Artisan 命令

```bash
# sail artisan queue:work
```

### 执行 Node/NPM 命令

```bash
# sail node --version

# sail npm run dev
```

### 容器 cli

```bash
# sail shell

# sail root-shell
```

### 执行 tinker

```bash
# sail tinker
```
