---
layout: post
title: "Git and Github"
date: 2017-09-09 23:31:00
tags: 版本控制 git github

---

The Difference Between Git and Github.

## 初识git and github
相信对于程序员来说，git 和 github 一定不会陌生，可能很多人每天都会与他们打交道。有次与同事聊
天，他问我，git 和 github 有什么区别呢。最开始，我的理解是，把 git 看成是一个版本控制工具，
而 github 作为一个源代码托管平台，我们可以通过 git 工具，从代码托管平台 clone 或者 commit
文件。

对于这个理解，确实显得过于肤浅，但是要谈到这两者之间的关系，还是需要从 git 的诞生历史说起。

## Git
Git 是一个分布式版本控制系统，最初是由 linux 之父 linus torvalds[^1] 于 2005 年以 GPL
发布的。 git 可以说是 linus 在完成 linux 这一宏伟产品后又一个划时代的产品，虽然他本人曾很谦
虚的说，如果没有 git，肯定会有 git 类 (git like) 的产品出现 [^2][^3]。

2002年至2005 年， Linus 顶着开源社区精英们口诛笔伐的压力，选择了一个商业版本控制系统
 BitKeeper 作为 Linux 内核的代码管理工具。BitKeeper 不同于 CVS 和 SVN 等集中式版本控制
 工具，而是一款分布式版本控制工具，它满足了 Linux 内核开发的技术需求。

分布式版本控制系统最大的反传统之处在于，可以不需要集中式的版本库，每个人都工作在通过克隆建立的
本地版本库中。也就是说每个人都拥有一个完整的版本库，查看提交日志、提交、创建里程碑和分支、合并
分支、回退等所有操作都直接在本地完成而不需要网络连接。每个人都是本地版本库的主人，不再有谁能提交
谁不能提交的限制，加上多样的协同工作模型（版本库间推送、拉回，以及补丁文件传送等）让开源项目的
参与度有爆发式增长。

但是 BitKeeper 毕竟是一款商业软件，这与开源精神不符，开源社区中很多人对此举提出了质疑，而且
 BitMover 公司也只是暂时将 BitKeeper 免费提供给 linux 内核团队使用。这中间发生了一个小插
 曲，在2005年4月，Andrew Tridgell（即大名鼎鼎的 Samba 的作者）试图对 BitKeeper 进行反向
 工程，以开发一个能与 BitKeeper 交互的开源工具。这激怒了 BitKeeper 软件的所有者 BitMover
 公司，要求收回对 Linux 社区免费使用 BitKeeper 的授权 [^4]。

但是当时现有的一些版本控制工具比如 CVS，或多或少都存在一些问题，特别是性能问题， linus 是极力
反对使用这些软件的，他甚至宁愿使用 diff 和 patch。在 linux 内核团队与 BitMover 公司交涉无
果的情况下，linus 决定开发一个版本控制工具，两周之后，git 诞生了。

与 SVN 和 CVS 等软件不同的是，Git 更关注文件的整体性是否有改变，Git 更像一个文件系统，它允
许开发者在本地获取各种数据，而不是随时都需要连接服务器。Git 的最大的特点就是离线分布式代码管
理，速度飞快，适合管理大型项目，难以置信的非线性分支管理。

git 从发布起就获得了迅速的推广，2005 年 4 月 3 日开始开发，4 月 6 日项目发布，而在 6 月 16
 日 Linux 内核 2.6.12 发布时， Git 已经在维护 Linux 核心的源代码了。

2005年7月26日，Linus功成身退，将 Git 的维护交给另外一个 Git 的主要贡献者 Junio C Hamano，
直到现在 [^5]。 git 虽然是在 linux 平台下开发的，但是显然，它已经可以跨平台运行在多个主流的
操作系统上。

## Github
GitHub平台于2007年10月1日开始开发。网站于2008年2月以beta版本开始上线，4月份正式上线。它是
一个通过Git进行版本控制的软件源代码托管服务，由GitHub公司（曾称 Logical Awesome）的开发者
 Chris Wanstrath、PJ Hyet t和 Tom Preston-Werner 使用 Ruby on Rails 编写而成。

**不同于开源的 Git，Github 提供了一套商业的解决方案。** GitHub 同时提供付费账户和免费账户。
这两种账户都可以创建公开的代码仓库，但是付费账户还可以创建私有的代码仓库。根据在 2009 年的 Git
用户调查，GitHub 是最流行的 Git 访问站点。除了允许个人和组织创建和访问保管中的代码以外，
它也提供了一些方便社会化共同软件开发的功能，即一般人口中的社区功能，包括允许用户追踪其他用户、
组织、软件库的动态，对软件代码的改动和bug提出评论等。GitHub 也提供了图表功能，用于概观显示
开发者们怎样在代码库上工作以及软件的开发活跃程度。[^6]

**回到我们最开始的那个问题，Git 是由 linus 开发的一款用于代码版本控制的分布式版本控制工具，
是基于 GPL 开源的；而 Github 是基于 git 的一种网站式的代码托管服务，提供了一种商业解决方案，
提供免费用户和付费用户两种模式。可以通过 git 访问 github 站点，在 github 上托管自己的源代
码。当然，github 还有很多其他的功能，可以到 github 上去看看。**

**参考文献：**

[^1]: [linus 一生只为寻找欢笑](http://mp.weixin.qq.com/s?__biz=MjM5ODQ2MDIyMA==&mid=403228545&idx=1&sn=f2d76137297fd33ed6e80126a0b67aa2&scene=21#wechat_redirect)

[^2]: [10 Years of Git: An Interview with Git Creator Linus Torvalds](https://www.linuxfoundation.org/blog/10-years-of-git-an-interview-with-git-creator-linus-torvalds/) <br>

[^3]: [Git 10 周年访谈：Linus 讲述背后故事](http://blog.jobbole.com/85772/)

[^4]: Git权威指南，第一章，1.4小节：Git—Linus 的第二个伟大作品

[^5]: [Git desc on wikipedia](https://en.wikipedia.org/wiki/Git)

[^6]: [GitHub desc on wikipedia](https://wc.yooooo.us/wiki/GitHub)
