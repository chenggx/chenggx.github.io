---
title: git 撤销相关操作
date: 2020-09-12 22:22:22
author: chenggx
top: false
categories: git
img: http://static.xiangdangnian.net.cn/blog/2020/09/12/19-33-32-a75c69a1c097a6ae4fffdf88b69eb2dc-b631d4.jpg
tags:
  - git
---

每次使用 git 需要进行版本回退相关的操作都要在搜索引擎重新查询相关命令，很是费时间，今天有空总结一下，算是记笔记方便以后使用。


## 撤销本地当前所有修改

```bash
git reset --hard
```

如果本地文件修改得一团乱，但是还没有 commit，可以通过这个命令恢复到上次 commit 的状态。

## 丢弃 commit 

```bash
git reset --hard commitID
```

将代码恢复到指定的 commitID 处。如果不指定 --hard 参数，那么不会改变工作区的文件。（一般使用都会带上该参数）

## 撤销 commit

```bash
git revert HEAD
```

commit 代码以后，你突然意识到这个 commit 有问题，应该撤销掉，这时执行该命令就可以了。该命令会新生成一个 commit 提交记录，并不改变其他内容，在 log 中可查看到撤销记录。基本属于撤销操作的首先方案。

> 该命令还有两个常用参数

- --no-edit：执行时不打开默认编辑器，直接使用 Git 自动生成的提交信息。
- --no-commit：只抵消暂存区和工作区的文件变化，不产生新的提交。


## 撤销 add 操作

```bash
git reset HEAD [filename]
```
可选参数

filename 指定撤销的文件。（不使用该参数表示所有已经 add 了的文件全部取消 add 状态）

## 替换上一次 commit 

```bash
git commit --amend -m "new Message"
```

如果我们在 commit 之后发现信息写错了，这是可以使用该命令修改信息。

**这时如果暂存区有发生变化的文件（也就是有文件被 add 了），会一起提交到仓库。所以，--amend不仅可以修改提交信息，还可以整个把上一次提交替换掉。**


## 暂时将未提交的变化移除，稍后再移入
$ git stash
$ git stash pop

如果我们在修改了一些文件后发现不应该在这个分支上操作，那么我们可以使用 git stash 将当前工作区的文件保存一下，然后切换的新分支，执行 git stash pop 将保存的内容恢复到新分支。


## 时光机

```bash
git reflog
```

只要HEAD发生了变化, 就会在reflog里面看得到。

配合 git reset 命令可以将代码切换到任意状态。


## 取消已经 push 的行为（强制 PUSH）

```bash
# 查询 git 所有变化日志
git reflog 

# 本地仓库回退到某一版本
git reset -hard xxxx

# 强制 PUSH，此时远程分支已经恢复成指定的 commit 了
git push origin master --force
```

![程序员的艺术人生](http://static.xiangdangnian.net.cn/4vB3ZRunrw8VzSW6cLpJyaibKfYTNCU0.png)