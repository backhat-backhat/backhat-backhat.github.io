---
layout: post
title: "libevent ios 锁屏后出现bug"
date: 2019-05-06 21:30:00
tags: ios libevent
---
libevent 在ios平台出现的怪异行为

	libevent是一个跨平台异步事件库，并且有http功能，比较健壮，
所以使用它开发http服务器会很方便。
	由于需要在移动端写一个http本地服务器作为代理服务器，
本身libevent我们app已经有使用到，所以决定使用libevent的evhttp
工具库，当然也可以使用其他的库或者自己写，libevent已经足够使用，
站在巨人肩上是一件非常好的事情，无需重造轮子，毕竟人的时间是有限的。
	上面是背景，写完之后，却发现ios平台有莫名奇妙的bug，按照下面
的步骤会出现不再接受新的客户端连接，经过libevent的官方回答和自己的
实验，发现这是ios的平台问题。
	1. 打开app(app初始化会启动http本地代理服务器, 监听某个端口)
	2. 锁屏(等待app休眠, 有些app需要几分钟, 有些则马上休眠)
	3. 解锁(app休眠后才能解锁)
	4. 客户端请求连接
	5. app没有任何响应

	ios解决方案(可选其中一个)：
	1. 自己写socket，写一个裸的http服务器，不用异步事件
	2. app休眠时，把http服务器关闭，app解锁休眠，重启http服务器

	macos等待测试