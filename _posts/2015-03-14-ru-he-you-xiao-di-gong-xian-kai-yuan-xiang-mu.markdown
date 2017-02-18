---
title: 如何有效地贡献开源项目
date: '2015-03-14 07:14:08'
tags:
- 搬运
---

为开源项目做贡献有许多好处. 当你贡献一个项目时, 你能学习到许多大型组织所实行的开发方式. 这些技能对于一个人的职业生涯有着很大的帮助, 并且能学习到很多在学校/工作中可能学习不到的知识.

本文面向那些初级或者中级的开发者, 为他们开始贡献开源项目提供帮助和指导. 我会使用 [Mozilla Foundation](https://mozilla.org/) 的 [Webmaker](https://webmaker.org/) 项目作为例子, 来告诉你如何有效地贡献开源项目.

### 学习案例: Mozilla Webmaker

我们将会通过参与到一个项目中, 了解贡献项目的整个流程, 在过程中我们将会使用到 [Github] (https://github.com/) 账号并且使用到 Git 以及使用到 Bugzilla. 本教程会向你展示如何将你修改的补丁合并到一个项目的主干. 我将会使用一个在我刚开始贡献开源项目时修复过的 bug 作为例子.

首先, 你必须找到项目的仓库. 在这个例子中, 我们将在 Webmaker 项目的一个名为 Profile 的组件下开始我们的工作.

```
https://github.com/mozilla/webmaker-profile-2
```

登录到你的 Github 账户, *Fork* 该项目. 在此之后, 在你的账户下你会看到你刚才 fork 的项目.

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_75%20Feb.%2004%2002.28.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_75%20Feb.%2004%2002.28.jpg)

在这个例子中, 我得到了以下仓库 URL:

```
https://github.com/tanay1337/webmaker-profile-2
```

你需要在你的系统中安装 Git. 你可以阅读 [如何安装 Git](http://git-scm.com/book/en/v2/Getting-Started-Installing-Git) 来完成 Git 的安装.

在你的 Github 仓库页面的右下角, 你会看到以下输入框:

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_76%20Feb.%2004%2002.30.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_76%20Feb.%2004%2002.30.jpg)

你可以复制输入框中的 URL 来将该仓库克隆 (clone) 到本地. 现在, 在命令行执行以下 Git 命令:

```
git clone https://github.com/tanay1337/webmaker-profile-2.git
```

这样做将会把仓库的代码导入到你的系统下的一个名为 *webmaker-profile-2* 的目录. 贡献指引文档通常都是存放在名为 `CONTRIBUTING.md` 的文件中, 同时项目介绍会在 `README.md` 中. 请仔细阅读这两篇文档. 文档包含了对开发者来说十分重要的信息.

### 进入 Bugzilla!

现在, 你必须得发现一些有意义并且简单的 Bug. 对于 Mozilla 相关的项目, 你可以使用 [Bugs Ahoy](http://www.joshmatthews.net/bugsahoy/) 来找到一些具有指导性并且在你能力范围之内的 bug. 和处理新特性类似, Mozilla 使用 [Bugzilla](https://bugzilla.mozilla.org/) 来管理 bug. 你可以使用 [Persona](https://login.persona.org/about) 登录到 Bugzilla. 当你发现一个相关 bug 时, 你应该评论该 bug, 并且表示你有兴趣修复它.

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_77%20Feb.%2004%2002.33.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_77%20Feb.%2004%2002.33.jpg)

同样地, 你可以从 Mozilla 的 [IRC](https://wiki.mozilla.org/IRC) 系统中寻求帮助, 帮助你得到该 bug 所在的具体源码文件. 他们是一群非常友善的团体, 乐于为你解决的第一个 bug 提供帮助. 万一没有人在线, 你可以尝试增加一个 `needinfo` 标记, 告诉该 bug 的指导者, 他将会帮助你.

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_79%20Feb.%2004%2002.36.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_79%20Feb.%2004%2002.36.jpg)

### 保存修改到本地仓库

现在, 假设你已经修复了 bug 并且在本地修改了几个文件, 你会需要在版本控制系统中看到那些你修改的文件的变化. 你只需要在 `webmaker-profile-2` 目录下输入以下命令.

```
git status
```

这将会展示那些修改过的或者新添加到本地仓库的文件的变化细节. 在你确认过文件的变化后, 将这些修改或者添加的文件放入暂存区 (staging area).

```
git add names_of_files
```

如果一切顺利, 你可以安全地提交这些文件.

```
git commit -m “Your message here"
```

### 清理提交日志并且推送变化

确认你提交的注释信息中没有包含多余的空格或是空行. 一种比较好的注释信息类似于 “Fixing Bug 1040556”, 原因我会在文章后面解释. 项目的维护人员更倾向于每一个 pull request 只包含一个提交. 所以, 如果你本地有多于一个的提交时, 你应该执行衍合 (rebase) 命令.

```
git rebase -i HEAD~2
```

以上命令假设你有两个提交, 使用 `-i` 参数表示打开交互式衍合 (rebase)模式. 这样做将会在屏幕上显示两次提交和提交时所带的注释信息, 注释信息上会带有 `pick` 字样的前缀. 只需要将你认为更好的提交注释前的 `pick` 字样用 `squash` 替换. 下一步两个提交消息将会被合并在一起.

恭喜你, 你已经成功衍合 (rebase) 了提交. 现在你只需要把变化推送到 Github 仓库.

```
git push
```

如果你已经推送了第一个提交并且在之后做了衍合 (rebase) 操作, 尝试使用以下命令.

```
git push -f
```

### 创建一个 Pull Request

现在打开你在线的仓库并且点击 **Pull Request** 按钮创建一个新的 pull request.

Pull request 会自动将提交的注释信息填入标题并且显示文件的变化对比.

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_49%20Jan.%2017%2013.47.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_49%20Jan.%2017%2013.47.jpg)

恭喜你, 你已经创建了你的第一个 pull request. 但你还需要做一些事情. 复制 pull request 的链接并且打开你在 Bugzilla 上的 bug. 选择 **Add an attachment** 并且将 pull request 的链接粘贴到这里. 选中名为 patch 的选择框并且增加 review 的标志.

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_80%20Feb.%2004%2002.38.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_80%20Feb.%2004%2002.38.jpg)

假设你得补丁是正确无误的, 你的指导者将你的 pull request 合并到仓库的主干并且该 bug 在 Bugzilla 上会自动标记为已解决. (只有当提交信息中包含 bug 号时才能自动标记).

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_81%20Feb.%2004%2002.39.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_81%20Feb.%2004%2002.39.jpg)

没有什么事比你看到你的代码被合并到主干并且被部署到有着上百万用户的网站上更令人热血沸腾了!

![https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_82%20Feb.%2004%2002.40.jpg](https://cms-assets.tutsplus.com/uploads/users/664/posts/23093/image/ScreenHunter_82%20Feb.%2004%2002.40.jpg)

我希望你能通过上述的步骤在 Webmaker 或者任何类似的开源项目上修复你第一个 bug.

附原文地址: [Effectively Contributing to Open Source Projects: Webmaker](http://code.tutsplus.com/tutorials/effectively-contributing-to-open-source-projects-webmaker--cms-23093)

### 后记

2015年, 给自己定了一个目标, 参与到一个开源项目中, 走出自己开发的小圈子. 这两天看到了这篇入门级的文章, 觉得颇有些帮助, 决定翻译一下. 一来试试本人的文笔是否可堪一读, 二来也希望能帮助到和我一样想迈出开源这一步的同学.

对于文章中段的 `git rebase` 的用法经过测试并没有得到原文中所描述的结果, 原因我会在后续研究后解释, 由于对整个流程并没有太大的影响, 因此我依然采用原文中的描述翻译.
