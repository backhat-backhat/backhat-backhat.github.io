---
layout: post
title: "redis源码分析 - 编程技巧总结"
date: 2016-10-27 23:30:00
tags: redis 编程 c语言 宏 变长数组
---
从redis源码中学习到的编程技巧总结

# 宏的用法
    
    #define VERSION "6.0.1"
    #define DATETIME　"datetime"
    
    //将变量 s 以字符串的形式输出
    #define xstr(s) __str(s)
    #define __str(s) #s
	
	//使用宏拼接字符串
    #define ECHO_STR ("jemalloc-" xstr(hello) "." xstr(world) "." xstr(version))
	//printf ("%s\n", ECHO_STR); 将输出 jemalloc-hello.world.version

	//同时，还可以使用如下方式拼接字符串
	char* buf = VERSION DATETIME;
	printf ("%s\n", buf); //将输出buf的值为 6.0.1datetime
	
上述代码中，为什么需要使用两个嵌套宏函数来完成字符串是输出呢，看如下例子就能知道结果：

	printf ("%s\n", xstr (VERSION)); //输出 "6.0.1"
	printf ("%s\n", _xtr(VERSION));  //输出 VERSION
	
后者不会对参数进行宏替换，直接当做字符串输出，前者才对参数进行宏替换

# 零长数组和变长数组
## 零长数组
在 GNU c 中允许零长数组，在 c++ 和 ANSI C 中都不允许使用，如下所示：

    typedef struct sds {
    	unsigned int len;
    	unsigned int free;
    	char buf[0];
    };
	
此结构体中包含一个长度为零的数组，但是它的使用必须满足一定的条件。

* 结构体中至少有一个其他类型的成员，否则会报错 "`error: flexible array member in otherwise empty struct`"
* 必须放在结构体的成员中的最后一个位置，否则会报错 "`error: flexible array member not at end of struct`"

**查看该结构体的大小发现，长度为 0 的数组长度为 0， sizeof (sds) == 8**

在 ISO C99 中，使用变长数组也可以实现同样的功能。
## 变长数组
    
    typedef struct sds {
    	unsigned int len;
    	unsigned int free;
    	char buf[];
    }sds;
	sds mSds = {5, 10, {'h', 'e', 'l', 'l', 'o'}};	//此结构体的变量只能在函数体外部定义和初始化，否则会报错
	//error: non-static initialization of a flexible array member
	//error: (near initialization for 'mSds')
	
通过 sizeof 查看大小
    
    sizoef (sds) : 8
    sizeof (mSds): 8
	
变长数组是不完全数据类型，不能使用 sizeof 获取它的大小。

**这种C99中伸缩数组(flexible array)，是对结构体功能的扩展。在结构体的原型申明时，可以申明一个没有指定数组长度的数组，在使用是，通过malloc动态决定结构体变量的数组大小。**

在 C 中申明一个数组

	int arr[NUM];
	
NUM的值一般是一个常量，且是不变的，但是在边长数组的长度可以在运行时指定。如下所示：

	int i;
	scanf ("%d", &i);
	int arr[i];
	
上述的数组长度就是在运行时决定的，这种类型的数组就称之为变长数组。

> **可参考文档 <br>
> https://gcc.gnu.org/onlinedocs/gcc-4.6.2/gcc/C-Extensions.html#C-Extensions**

    sds * pSds = (sds*)malloc (sizeof (sds) + length + 1);
    pSds->len = length;
    pSds->free = 0;
    memcpy (pSds->buf, str, length);
    pSds->buf[length] = '\0';
	
在 redis 源码中，sdsnew 函数每此创建一个结构体后，都是返回的 pSds->buf 进行处理，这样，结构体往后偏移了 sizeof(struct sds)个字节（即 8 个字节），所以每次在对 buf 处理的时候，先转成 sds 结构体时，都需要往前偏移 8 个字节，得到 len 和 free。

	void sdslen (void * s) 
	{
		struct sds * sh = (void *)(s - (sizeof (struct sds)));
		return sh->len;
	}