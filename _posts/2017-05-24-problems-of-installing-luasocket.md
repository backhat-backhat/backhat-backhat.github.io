---
layout: post
title:  "luasocket的安装和部署"
date:   2017-05-24 10:18:00
tags: luasocket
---
本文主要介绍 luasocket 的安装和部署

初学 lua，当读到《lua 程序设计》中9.4节非抢占式(non-preemptive)多线程时，例子中用到了

	require "socket"
	
在我的 lua 环境中，直接使用出错了，因为这是一个外部c的动态库，需要下载编译配置好才能使用。

luasocket 是Diego Nehab 写的，GitHub的地址为 [https://github.com/diegonehab/luasocket]，现在的版本较之前的版本做了一些改动。下面说一下编译和安装的期间的一些问题： <br>
1、 在根目录下有一个 makefile 文件，查看 makefile 文件，发现其实都是进入 src 目录后在进行编译的，具体的编译信息进入 src 中查看 makefile 文件

2、 在src中，makefile 文件显示，luasocket 能够支持的平台有 linux, win32, Solaris, mingw, macosx。如果你是在 linux 中的话，

	make linux
	
即可。但是需要注意，这里依赖需要有 lua.h 等头文件，你需要在 makefile 中修改一下

	LUAINC_linux
	
这个变量的值，即将路径修改为你主机上的目录，不然编译会出错

3、 编译成功后，不会像之前版本中直接编译成功为 socket.so 这样的库文件，生成的库文件为

	mime-1.0.3.so
	unix.so
	serial.so
	socket-3.0-rc1.so
	
当然，主要关注前两个即可。使用

	make install
	
安装，(这里说明一下，楼主的 lua 是 5.1 版本的)，会安装在

	/usr/local/share/lua/5.1/
	/usr/local/lib/lua/5.1/
	
这两个目录中，查看 makefile 文件也能发现

	install: 
		$(INSTALL_DIR) $(INSTALL_TOP_LDIR)
		$(INSTALL_DATA) $(TO_TOP_LDIR) $(INSTALL_TOP_LDIR)
		$(INSTALL_DIR) $(INSTALL_SOCKET_LDIR)
		$(INSTALL_DATA) $(TO_SOCKET_LDIR) $(INSTALL_SOCKET_LDIR)
		$(INSTALL_DIR) $(INSTALL_SOCKET_CDIR)
		$(INSTALL_EXEC) $(SOCKET_SO) $(INSTALL_SOCKET_CDIR)/core.$(SO)
		$(INSTALL_DIR) $(INSTALL_MIME_CDIR)
		$(INSTALL_EXEC) $(MIME_SO) $(INSTALL_MIME_CDIR)/core.$(SO)
		
这里有一点不同的是，安装的时候，将 `unix.so` 和 `mime-1.0.3.so` 命名为 `core.so`，因为在 socket.lua 文件中，使用的是 `require "socket.core"`

#require 是如何查找库文件的呢
查看 lua 手册 [http://www.lua.org/manual/5.2/manual.html#6.3]

lua 中，package 库为加载模块提供了所有基础方法，所有信息保存在表 package 中。

	require(modulename)
	
首先，lua会先在 `package.loaded` 表中查找是否该模块已经加载，如果已经加载过了，直接返回保存在该表中的信息 `package.loaded[modulename]`。如果没有，lua 会试图为该模块找一个加载器(loader)，首先在 `package.preload` 表中查找模块，是否有 `package.preload[modulename]`，如果有，就以该函数作为加载器。如果没有，lua 会尝试从 lua 文件或者 c 文件中加载该模块。根据 `package.path` 查找 lua 文件，否则，继续根据 `package.cpath` 查找 c 文件。

Lua 用于搜索 lua 文件的路径存储在 `package.path` 中，lua 启动之后，就会以环境变量 LUA_PATH 来初始化这个变量，如果没有找到这个环境变量，那么会使用编译时定义的默认路径来初始化。

当Lua无法找到与模块名相符合的 lua 文件时，就会查找 c 程序库。查找c程序库的路径存储在 `package.cpath` 中，这个变量视通过环境变量 LUA_CPATH 来初始化的。

所以，当在环境中安装好 luasocket 的时候，使用
	
	require "socket"
	
仍然出现 

	./socket.lua:12: module 'socket.core' not found
	
的错误时，在之前先配置一下 `package.path` 和 `package.cpath` 即可

	package.path = "/usr/local/share/lua/5.1/?.lua"
	package.cpath = "/usr/local/lib/lua/5.1/?.so"