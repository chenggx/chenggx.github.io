---
title: python常用虚拟环境工具
date: 2024-03-06 20:23:06
author: chenggx
categories: python
tags:
 - python
  
---


## 什么是虚拟环境以及为啥要是用虚拟环境

Python 虚拟环境是一种用于隔离不同项目所需 Python 版本和第三方库的工具，通过虚拟环境，您可以在同一台计算机上创建多个独立的 Python 环境，每个环境都可以拥有不同的 Python 版本和安装的库，从而 避免不同项目之间的版本冲突。

![Python 虚拟环境概念图](https://static.xiangdangnian.net.cn/blog/01-flowchart-venv-concept.png?v=2)

## 常见的虚拟环境工具

> 不用所有的工具都会熟练是用，但每种工具都要了解，以确保看到第三方项目可以知道使用了那种工具，并可以进行是用。

- venv
- virtualenv
- pipenv
- conda（本人是新手，先不了解了）

### venv

venv 是 python 自带的一种虚拟环境工具，以下有几点需要注意：

1. venv 没法创建不同版本的 python 环境，也就是 python3.6 没法创建 python3.7 的虚拟环境。
2. venv 是 python3.3 版本引入的，所以 python3.3 以下的版本无法通过 venv 创建虚拟环境。

#### 使用方法

1. 在项目目录中使用 `python -m venv 目录名` 创建虚拟环境。创建完成后会在当前目录中创建出一个虚拟环境目录

```bash
# mkdir test && cd test

# python -m venv myvenv

# ll
total 0
drwxr-xr-x  6 cc  staff   192B  3  6 16:19 myvenv
```

2. 激活虚拟环境，激活后 bash 就是指定版本的 python

```bash
# source myenv/bin/activate
(myvenv)#
```

3. 安装依赖包

```bash
(myvenv)# pip install package_name
```

4. 退出虚拟环境

```bash
(myvenv)# deactivate
#
```

![venv 操作流程图](https://static.xiangdangnian.net.cn/blog/02-flowchart-venv-workflow.png?v=2)

### virtualenv

virtualenv 是一个第三方的虚拟环境工具，可以创建 python2 和 python3 的虚拟环境。

#### 使用方法

1. 安装 virtualenv

```bash
# pip install virtualenv
```

2. 创建

```bash
# mkdir test && cd test
# virtualenv myenv
# ll
total 0
drwxr-xr-x  6 cc  staff   192B  3  6 16:19 myvenv
```

3. 激活

```bash
# source myenv/bin/activate
(myvenv)#
```

4. 安装

```bash
# pip install package_name
```

5. 退出

```bash
(myvenv)# deactivate
#
```

### pipenv

pipenv 也是一个第三方的虚拟环境工具，可以创建 python2 和 python3 的虚拟环境。激活系统后会在系统目录下生成虚拟环境目录，在当前目录下生成一个 pipfile 和 pipfile.lock 文件。类似有 php 中的 composer.json 和 composer.lock。可以不用是用 requirements.txt 来管理依赖包了。

#### 是用方法

1. 安装工具

```bash
# pip install pipenv
```

2. 创建环境
   通过 `pipenv --venv` 这个命令可以查看虚拟环境的目录位置

```bash
# mkdir test && cd test
# pipenv install
# ll
total 8
-rw-rw-r-- 1 admin admin 138 Mar  6 16:41 Pipfile
-rw-r--r-- 1 admin admin 453 Mar  6 16:41 Pipfile.lock
```

3. 安装依赖包
   安装完成后会在 Pipfile 和 Pipfile.lock 中写入

```bash
#  pipenv install flask

// 是用如下命令安装开发所依赖的扩展
# pipenv install --dev requests
```

4. 进入虚拟环境
   不建议在该环境中进行 pip 扩展的安装，因为无法写入 pipfile 中。

```bash
# pipenv shell
```

5. 退出虚拟环境

```bash
# exit
```

6 删除虚拟环境

```bash
# pipenv --rm
```

7. 指定虚拟环境的 python 版本

```bash
# pipenv --python /usr/bin/python2.7
```

![pipenv 对比图](https://static.xiangdangnian.net.cn/blog/03-comparison-pipenv-comparison.png?v=2)

#### pipfile 示例

```
[[source]]
url = "https://pypi.org/simple"
verify_ssl = true
name = "pypi"

[[source]]
url = "https://pypi.tuna.tsinghua.edu.cn/simple/"
verify_ssl = true
name = "pypitunatsinghuaedu"

[packages]
flask = {version = "*", index = "pypitunatsinghuaedu"}

[dev-packages]
requests = {version = "*", index = "pypitunatsinghuaedu"}

[requires]
```
