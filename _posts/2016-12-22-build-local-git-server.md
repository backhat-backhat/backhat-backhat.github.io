---
layout: post
title:  "搭建本地git服务器!"
date:   2016-12-22 10:18:00
tags: git 搭建
---
讲述在局域网范围内搭建 git 服务器的方法

搭建局域网范围内的 git 服务器，LZ模拟实验，使用了三个虚拟机，一个 RHEL6，两个 Ubuntu server

**一、 安装git 和 ssh服务，LZ使用的协议是 ssh** <br>
在 RHEL 系统中，我建的是本地仓库，在 root 用户下执行

	yum install git
在 Ubuntu 下使用本地仓库

	sudo apt-get install git

**二、 创建远程 repository**  <br>
LZ 选的远程 git 服务器是 Ubuntu server 系统，首先在 root 用户下创建一个 git 用户

	useradd -m git
	passwd git
注意，`-m` 选项在创建用户的同时会为该用户创建 home 家目录。passwd 为该用户设置密码

在家目录中创建 repo

	mkdir myRepositories.git
	cd myRepositories.git
	git --bare init

<p style="font-family:consolas;color:blue">
注意： 此处最好使用 git --bare init，不要直接使用 git init。git init 初始化，会在远程仓库的目录下包含 work tree，当本地仓库向远程仓库 push 时，如果远程仓库正在 push 的分支上，那么 push 的结果就不会反应在 work tree 上，但是通过 git log 能够查看到提交的日志，知道提交是否成功。使用 git reset --hard 就能看到 push 之后的内容了。
</p>

__服务器端使用 `--bare` 初始化 repo，不需要 checkout 出文件，如果想查看服务器上 push 的文件，可以通过`git log` 查看日志，也可以通过 `git show-branch -a --color=always` 的方法查看__

好了已经创建好了远程服务器的 repo

**三、 在本地操作**  <br>
选一台主机(虚拟机)作为远程本地主机，将远程服务器上的 repo 克隆 (clone) 下来 <br>

* 因为此处使用的是 git@host_ip:directory 的形式 clone 远端的 repo，使用的是 ssh 协议，在这里需要创建公钥和私钥来连接远端的主机，可以设置密码，也可以采用无密码登陆的方式。

	ssh-keygen -t rsa -b 4096 -C "comments"

然后选择路径 Enter file in which to save the key，直接回车选择默认路径，ssh 服务一般是将公钥和私钥文件存放在 $HOME/.ssh 目录中

Enter passphrase 是让你设置密码，直接回车将密码设置为空，可以采用无密码登陆。

* 创建好后，在 $HOME/.ssh 目录中可以发现多了两个文件 `id_rsa` 和 `id_rsa.pub` ，前者是私钥，后者是公钥，然后将 `id_rsa.pub` 文件通过 scp 发送到远端主机上

登陆远端主机，进入到 $HOME/.ssh 目录中，查看目录中是否存在 authorized_auth 文件，如果没有就新建一个，然后将发送过来的 `id_rsa.pub` 文件中的内容追加到该文件中

	cat id_rsa.pub >> authorized_auth
* 回到本地机器，新建一个目录 myGitRepo，现在就可以像在 github 上一样执行 git 的各种命令了。

clone fetch pull push

	cd myGitRepo
	git init	//
	git status
	git clone git@host_ip:/home/git/myGitRepositories.git
注意： <br>
`git@host_ip:/home/git/myGitRepositories.git`中， 是指 host_ip 主机上 git 用户下，家目录中的 myGitRepositories.git 目录，一般是相对于家目录的相对位置。

具体 git 的使用方法可以参考如下文章，也可以查看 `gittutorials`

从0开始学习 GitHub：__http://blog.csdn.net/column/details/13170.html__ <br>
Git详解，服务器上的Git：__http://blog.csdn.net/hustpzb/article/details/7287954__