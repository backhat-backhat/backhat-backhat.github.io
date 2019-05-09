---
layout: post
title: "面试总结1"
date: 2017-03-15 23:30:00
tags: 面试 redis 字节序
---
面试总结

## redis list 是否有长度限制 <br>
list 的数据结构为

{% highlight ruby %}
	typedef struct listNode {
	    struct listNode *prev;
	    struct listNode *next;
	    void *value;
	} listNode;
	
	typedef struct list {
	    listNode *head;
	    listNode *tail;
	    void *(*dup)(void *ptr);
	    void (*free)(void *ptr);
	    int (*match)(void *ptr, void *key);
	    unsigned long len;
	} list;
{% endhighlight %}

再看下获取长度 len 的宏函数定义

{% highlight ruby %}
	#define listLength(l) ((l)->len)	
	
	//llen command 的实现函数
	unsigned long listTypeLength(robj *subject) {
	    if (subject->encoding == REDIS_ENCODING_ZIPLIST) {
	        return ziplistLen(subject->ptr);
	    } else if (subject->encoding == REDIS_ENCODING_LINKEDLIST) {
	        return listLength((list*)subject->ptr);
	    } else {
	        redisPanic("Unknown list encoding");
	    }
	}
{% endhighlight %}

可知，list 的长度是通过 list 结构中的长度字段 len 直接返回的，这是一个 `unsigned long` 类型的变量，说明 list 的长度是有大小限制的，就是 `unsigned long` 所能表示的最大值。

## 字节序的由来 <br>
这个问题，初始的时候，问的我有点懵，“字节序的由来”，这段历史我可真是不大了解，要是问我字符编码的历史，我还能说说。我说了一下大小端和网络数据传输的问题，面试官说这是果，不是因。好了，废话扯这么多，直接入主题。

先扯扯历史。

大家都知道，大多数计算机使用的都是8位的字节，作为最小的可寻址的存储器单位，而不是直接在存储器中访问单独的位。存储器的每一个字节都有一个唯一的数字标识，这就是地址。

对于跨越多字节的程序对象，我们必须建立两个规则：这个对象的地址是什么，以及在存储器中如何排列这些字节。一般多字节对象都被排列为连续的字节序列，对象的地址为所使用的字节中的最小地址。

**在某些机器中选择存储器按照从最低有效字节到最高有效字节的顺序存储对象，但是另一些机器则正好相反。**前一种规则，称为“小端法(little endian)”，后一种称为“大端法(big endian)”。大多数 Intel 兼容的机器都采用小端法，而大多数 IBM 和 Sun Microsystems 的机器都采用的是大端法。

假设变量 X 的类型为 int，位于地址 ox100 处，int 有四个字节大小，地址是所使用字节地址最小的那个，所以这四个字节地址应该分别为 0x100, 0x101, 0x102, 0x103，假设 X 的 16 进制为 0x01234567，大小端表示，地址从小到大：

	地址：	0x100	0x101	0x102	0x103
	大端： 	0x01	0x23	0x45	0x67
	小端：	0x67	0x45	0x23	0x01
	
可以看出，大端法，更加接近于人类的习惯方式。(big-endian is how humans naturally process numbers)。

归根结底，字节序，是因为不同类型计算机内部数据存储的字节排列顺序不同产生的。

endian，端的由来，出自 Jonathan Swift 的《格列夫游记》中描述的异常荒诞的争论与冲突：在吃鸡蛋前，是应该先打破较大的一段还是较小的一端。Danny Cohen，以为网络协议的早起开创者，第一次使用大端和小端这两个术语来指代字节序(byte order)。

对于大多数应用程序员来说，机器使用的字节序顺序是完全不可见的，无论为那种类型的机器编译的程序都会得到同样的结果。但是，当不同类型的机器之间通过网络传输二进制数据时，不同类型的机器接收到的数据将会是反序，所以，网络应用程序的代码编写必须遵守已建立的关于字节顺序的规则，即网络数据传输都是大端方式，这样，网络应用程序无论是在发送还是在接收数据时，都会根据字节序的规则对数据进行转换，转换成它能够处理的字节顺序表示。

**参考资料**  <br>
1. 《深入理解计算机系统》，2.1.4节 寻址和字节顺序 <br>
2. [__how to teach endian__](http://blog.erratasec.com/2016/11/how-to-teach-endian.html#.WMlYvzhx0lB)