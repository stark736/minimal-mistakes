---
title: Git Submodule
date: '2014-09-08 01:40:37'
---

### 什么是Git Submodule

当你在一个项目上工作时，你也许会需要使用另外一个项目的库。举个例子，假设你在开发一个网站，为之使用 ORM 库。你不想编写一个自己的 ORM 库，而是使用第三方的 ORM 库。这时，你可能不得不像 Nodejs apm 或者 Ruby gem 那样包含来自共享库的代码，亦或者将代码直接拷贝到自己的项目树中。如果使用包含库的方法，那么定制这个库将会变得复杂。直接将代码拷贝到自己的项目中的问题是，当上游被修改时，任务你进行的定制化修改都很难被归并。

Git 通过子模块（Submodule）解决这个问题。子模块允许你将一个 Git 仓库当做另外一个 Git 仓库的子目录。这允许你克隆另外一个仓库到你的项目中并且保持你的提交相对独立。

### 创建子模块

```shell
$ git submodule add git@domain.com:another_project.git another_project
```
该命令会在你的项目仓库下产生一个 `.gitmodules` 文件（该文件也将处于版本控制之下），来记录你的 submodule 信息，保存了子模块项目 URL 和你拉取到本地的子目录，同时 another_project 项目也 clone 下来，此时观察你的 Git 仓库。
```shell
$ git status
# On branch master
# Changes to be committed:
#   (use "git reset HEAD ..." to unstage)
#
#       new file:   .gitmodules
#       new file:   another_project
```
你可能会注意到，git 只记录了 submodule 目录，而没有记录目录下的文件。实际上，git 是按照 commit id 来比对 submodule 变动的。
```shell
$ git add .gitmodules another_project
$ git commit -m "Add another_project submodule"
$ git submodule init
$ git push
```
`git submodule init` 将会初始化项目的本地配置文件。`git push` 将添加的子模块推送到远程仓库，以方便多人协作开发。

### 更新子模块

此时，你的团队中的其他成员需要与你协同开发包含子模块的项目。
```shell
$ git clone git@domain.com:another_project.git
```
当他们接受到这样的项目，他们将会得到包含子模块的目录，但里面没有文件。此时，必须运行两个命令：`git submodule init` 和 `git submodule update` 来初始化本地的子模块数据。`git submodule init` 上文已经介绍过。`git submodule update` 将会从子模块所属的远程仓库拉取所有数据，并检出上层项目里**所指向的子模块的提交**。
```shell
$ git submodule init
$ git submodule update
```
此时，another_project 远程仓库有数据更新，本地需要拉取远程的更新数据。
```shell
$ cd another_project
$ git pull <远程仓库> <远程分支>
```
亦或者
```shell
$ git submodule foreach git pull <远程仓库> <远程分支>
```
如果 `git pull <远程仓库> <远程分支>` 不指定远程仓库和分支，Git 会提示 `You are not currently on a branch` 的错误（默认子模块是属于游离状态 ‘detached HEAD’ state，不属于哪个分支，详情见子模块的坑）。现在，让我们查看下项目的状态。
```shell
$ git status
# On branch master
# Your branch is up-to-date with 'origin/master'.

# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)

# 	 modified:   another_project (new commits)

# no changes added to commit (use "git add" and/or "git commit -a")
```
刚才从远程仓库拉取并归并的仅仅**只是一个指向子模块的一个提交指针**，而并不是子模块目录中的代码。这时，你两个选择：在上层项目中提交子模块最新提交的指针；
```shell
$ git add another_project
$ git commit -m "Updated another_project"
$ git push
```
或者，将上层项目恢复到最后指向子模块的提交指针。
```shell
$ git submodule update
```

### 删除子模块

```shell
$ cd your_project
$ git rm --cached another_project
$ rm -rf another_project
$ vim .gitmodules
...remove another_project...
$ vim .git/config
...remove another_project...
$ git commit -a -m 'Remove another_project submodule'
$ git pull
```
这样，子模块就彻底与你的项目 say goodbye 了。
