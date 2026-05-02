---
layout: git
title: git常用命令总结
date: 2019-04-06 21:58:14
author: chenggx
img: http://static.xiangdangnian.net.cn/blog/d24925e626c0493d8e0e8f3c0bb6210c.jpg
top: false
category: git
tags:
 - git
---

> 内容不定期更新

## 自己常用的 git alias 可以在命令行中更好的显示 log 信息

```shell 
alias gl="git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"

alias gll="git log --graph --abbrev-commit --decorate --all --format=format:'%C(bold blue)%h%C(reset) - %C(bold cyan)%aD%C(dim white) - %an%C(reset) %C(bold green)(%ar)%C(reset)%C(bold yellow)%d%C(reset)%n %C(white)%s%C(reset)'"
```

## 1. git commit --amend

```git
git commit --amend
```
> 解释：当第一次 commit 之后，发现没有修改正确，但是又不想在 git log 中出现两次提交记录，那么在第二次修改后，可以使用 git commit --amend 命令进行修改提交。此时提交成功后就只有一条 commit 记录了。

## 2. gitignore 生效

> 解释: .gitignore 文件是忽略 git 项目中不需要提交的内容的配置文件。但是当某次提交的时候把不需要的内容提交到 git 版本控制中了之后。再想修改 .gitignore 文件来忽略那个文件的话是不会生效的。需要先想文件在 git 缓存中删除，然后在进行提交。就可以了。（下面是操作步骤）

```git
git rm -r --cached .   //如何是目录的话，增加 -r 参数
git add .
git commit -m 'fix ignore'

//删除指定文件的缓存
git rm --cached xx.html

```

## 3. git reset --soft/hard  **hash***
```git
git reset --soft  **hash***
git reset --hard  **hash***
```
> 解释：当执行 git commit 之后，想回到之前的版本，可以使用 git reset 进行版本回退。--sorf 参数的意思是，git log 撤销版本，但代码中的内容没有回退。 --hard 参数的意思是，git log 中的记录和 文件内容完全回退到选的的版本。（--hard 更常用）

## 4. git stash

```git
git stash           //将当前工作去内容进行暂存，然后就进行切换分支或其他的操作了
git stash list      //查看当前已经暂存的记录列表
git stash pop       //将当前暂存内容回复到工作区并删除暂存记录列表中的内容
git stash apply     //将当前暂存内容回复到工作区，不删除记录
```

## 5. git rebase (变基)
> 解释：当在两个分支进行操作的时候，一般我们使用 merge 将两个分支进行合并，但是这样会产生一个新的提交，并且另一个分支的提交记录在当前分支是不存在的。为了解决这个问题，可以使用 rebase。

```git
//前提：master 分支和 new-feature 分支都进行了多次提交

git checkout new-feature        //首先切换到新分支
git rebase master               //将 master 分支的提交记录添加到当前分支
git checkout master             //切换回 master 分支
git merge new-feature           //将 new-feature 分支合并入master，此时 master 分支中就有了 new-feature 分支的提交记录，但是不会产生新的提交记录。此命令可以使提交历史更为整洁。
```
