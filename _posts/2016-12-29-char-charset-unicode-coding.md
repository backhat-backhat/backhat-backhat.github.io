---
layout: post
title: "关于字符、字符集、unicode和编码"
date: 2016-12-29 21:41:00
tags: 字符 字符集 unicode 编码
---
本文主要解释一下字符、字符集、unicode 和编码的概念

# 历史背景
我们知道，在计算机内部，所有的信息都是通过二进制来表示的，一个字节有8比特，通过 0 和 1 表示的话，2<sup>8</sup>=256，就可以表示 256 中不同的状态。在上世纪 60 年代，美国制定了一套字符编码，成为 ASCII 编码，全称为美国标准信息交换代码(American Standard Code for Information Interchange)，32 以下的字符为_不可打印字符_，即控制字符，32以上为_可打印字符_。

但是 1 个字节能够表示 256 个符号，ASCII 码只有 128 个字符，那么高 128-255 位是否能够用于表示其他的符号？ <br>
答案是肯定的。 IBM-PC 发展出了后来叫做 OEM 的字符集，它支持某些欧洲语言中出现的方言字符，以及一堆画线用的符号，包括横线、竖线、右侧带小线条的横线等等。但是，这导致出现了一种 OEM 字符集乱像，各个不同地区都发展出了自己的 OEM 字符集，使得同一种语言中对高 128 个字符出现了很多种不同的解释。最终，这种 OEM 字符集乱像被 ANSI 标准化了。

在 ANSI 标准中，人们对地位的 128 个字符的含义打成了一致，与 ASCII 码相同，但根据地区的不同，对编号为 128-255 的字符处理方式也不同。这些子系统被称为代码(Page Code)。

但是在亚洲，尤其是我大中华，区区 256 个字符根本表示不了这么多的汉字。而通常解决这一问题的办法，使用一种叫做DBCS的杂乱系统，即双字节字符集(Double Byte Character Set)，有些字符使用 1B 存储，有些使用 2B 存储。但是这种乱象，使得计算机之间数据传输变得非常困难。幸运的是，Unicode 编码应运而生。尤其是互联网的兴起，使得 Unicode 编码得到了广泛的推广。

#Unicode
Unicode 是一个字符集，它几乎支持地球上所有自然的书写系统，甚至还支持某些杜撰出来的书写系统，如克林贡语(Klingon)。 <br>
Unicode 当然是一个很大的集合，现在的规模可以容纳 100 多万个符号。每个符号的编码都不一样，比如，U+0639 表示阿拉伯字母 Ain，U+0041 表示英语的大写字母 A，U+4E25 表示汉字"严"。前缀 U+ 表示的就是 “Unicode”，数字是十六进制的。具体的符号对应表，可以查询 www.unicode.org，或者专门的[汉字对应表](http://www.chi2ko.com/tool/CJK.htm)。

Unicode 能定义的字符并没有真正的上限，事实上 Unicode 的字符数量远超过了 65535(两个字节能表示的最大无符号整数)，并不是所有的 Unicode 字符都能使用 2B 来存储。

以上说的这些都是 Unicode 的编码的表示方式，更准确的说就是数字，但是 Unicode 在内存中到底如何存储的呢？

## Unicode 的编码
最早人们对 Unicode 编码的实现是每个数字都用 2B 存储，这就导致了“所有的 Unicode 字符都是 2B”的错误印象。于是，Hello 就变成了：

	00 48 00 65 00 6C 00 6C 00 6F
	
还能写成这样

	48 00 65 00 6C 00 6C 00 6F 00
	
最开始就是使用这两种高位补零和低位补零的方式来存储 Unicode 编码的，根据机器的 CPU 来选取处理速度更快的存储方式。（也就是大家熟知的 big endian 和 little endian 模式）。为了使两者有所区别，规定，在每个 Unicode 字符串前面插入 FE FF，这就叫做_Unicode字节顺序标记(Unicode Byte Order Mark)_。如果高地位互换，就变成了 FF FE。**而实际使用中的 Unicode 编码并不一定都带有开头的BOM**

> Unicode编码中表示字节排列顺序的那个文件头，叫做BOM（byte-order mark），FFFE和FEFF就是不同的BOM。 <br>
UTF-8文件的BOM是“EF BB BF”，但是UTF-8的字节顺序是不变的，因此这个文件头实际上不起作用。有一些编程语言是ISO-8859-1编码，所以如果用UTF-8针对这些语言编程序，就必须去掉BOM，即保存成“UTF-8—无BOM”的格式才可以，PHP语言就是这样。

## 问题
在 Unicode 之前，大部分存储的文档都是采用的 ANSI 和 DBCS 字符集，那么如果推广 Unicode 编码，之前那些文档的编码，谁来转换呢？**然后 UTF-8 出现了**

## UTF-8
UTF-8 是在互联网上使用最广泛的一种 Unicode 的实现方式。__注意，UTF-8 是 Unicode 的一种实现方式。__这只一套新的编码，即用8为的字节把由 Unicode 的编码，也就是那些神奇的 “U+数字” 的字符串存储在内存中的方法。

在 UTF-8 中，0-127 存储在 1B 中，只有 128 以上的编码采用 2B、3B甚至6B的方式来存储。

UTF-8的编码规则很简单，只有二条：

1）对于单字节的符号，字节的第一位设为0，后面7位为这个符号的unicode码。因此对于英语字母，UTF-8编码和ASCII码是相同的。

2）对于n字节的符号（n>1），第一个字节的前n位都设为1，第n+1位设为0，后面字节的前两位一律设为10。剩下的没有提及的二进制位，全部为这个符号的unicode码。

	Unicode符号范围 | UTF-8编码方式
	(十六进制) | （二进制）
	--------------------+---------------------------------------------
	0000 0000-0000 007F | 0xxxxxxx
	0000 0080-0000 07FF | 110xxxxx 10xxxxxx
	0000 0800-0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx
	0001 0000-001F FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
	0020 0000-03FF FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx 10xxxxxx
	
跟据上表，解读UTF-8编码非常简单。如果一个字节的第一位是0，则这个字节单独就是一个字符；如果第一位是1，则连续有多少个1，就表示当前字符占用多少个字节。

这么做有一个很方便的地方，就是英语文本在 UTF-8 中的表示和在 ASCII 中的表示完全相同，在 ANSI 以及其他所有 OEM 字符集中的表示都完全一致。当然，还有一个好处，就是某些旧版的代码会把值为 “0” 的字节当做字符串的 null(NULL TERM)结束符处理，而 UTF-8 编码不会错误的阶段字符串。

下面以汉字 "中" 为例，讲解 UTF-8 编码的转换： <br>
“中”的 Unicode 编码为 U+4E2D，在上面给出的 Unicode 范围中查询，发现 UTF-8 编码需要 3B 存储 <br>
![中转换成UTF-8编码](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/uft-8_code_translation.png)

到目前为止，一共介绍了三种 Unicode 编码方法。比较传统的是用2B存储每一个编码，也叫 UCS-2 (2代表2个字节)，或者 UTF-16，但是还是需要区分高位补零还是地位补零，还有一种是计较受欢迎的的 UTF-8 标准。

实际上，Unicode 还有许多其他的编码方法，有一种类似于 UTF-8 的编码方法 UTF-7，它的最高位始终是 0。还有一种为 UCS-4，使用4B来存储每一个编码，好处是每一个编码都对应相同的字节数，但是很浪费内存。

## 关于编码最重要的事实
> 不能确定编码的字符串时没有意义的。 <br>
> 所谓的纯文本，根本就不存在。

那么如何存储字符串所用编码类型的信息？有一个标准的做法，就是对于所有电子邮件消息，你需要在表头有这样一个字符串：

	Content-Type: text/plain; charset="UTF-8"
	
对于网页而言，以前 HTTP 协议的设计者想和电子邮件保持一致，要有服务器随网页本身返回一个 Content-Type 的 HTTP 报头。也就是说编码信息不存在于 HTML 文件内部，而是作为响应消息报头的一部分，先于 HTML 页面发送，那么程序或者浏览器就知道使用什么样的编码读取HTML页面的内容了。

## linux c 中输出中文
在 linux c 中我们使用 wchar_t ("款字符 wide character") 而不是 char 申明字符串，并且使用 wcs 函数族而不是 str 函数族，表示 UCS-2 字符串，只需要在前面加上一个L：

	L“hello”
	L"倚楼听风雨，划船听雨眠"
	
如下例子：

{% highlight ruby %}
	#include <stdio.h>
	#include <stdlib.h>
	#include <locale.h>
	#include <string.h>
	
	int main (int argc, char* argv[])
	{
		wchar_t wcs[] = L"倚楼听风雨";
		setlocale (LC_ALL, "en_US.UTF-8");	//通过 locale -a 查看
		
		wprintf (L"wide character: %ls\n", wcs);
		wprintf (L"wide character: %s\n", wcs);
	
		char ms[128];
		memset (ms, 0, sizeof (ms));
		wcstombs (ms, wcs, sizeof (ms));
		printf ("multbyte: %s\n", ms);
	
		return 0;
	}
{% endhighlight %}
	
上述代码输出结果为：
	
	wide character: 倚楼听风雨
	wide character: P
	
wprintf 使用的是宽字符流 (wide stream)，使用 %ls，wprintf 会将 wcs 看成是宽字符产，而 wcs 就是宽字符串，所以输出预期结果

	wide character: 倚楼听风雨
	
但是如果使用的是 %s，wprintf 会将对应的参数视为普通字符串 mbs，尽管我们还是很清楚它其实是个 wcs。wprintf 使用的是 wide stream,因此需要将所给的 mbs参数转换为 wcs 再由 wprintf 完成输出；这个转换是由 wprintf 隐式的对 mbs 不断调用mbrtowc来 完成，转换规则依然是和locale相关的。

我们知道 "倚" 的 Unicode 编码为 U+501A，那么 wcs 的内存布局为

	0x1a 0x50 0x00 0x00 0x7c 0x69 0x00 0x00
	
LZ的机器为 little endian

那么在 wprintf 将 wcs 转成普通的 mbs 后，输出按照字节输出，遇到 0 将作为 null 结束字符处理，所以输出 P。 0x1a 为控制字符，0x50 表示的大写字母P，后面的0x00 作为结束字符处理。

但是程序中，同时使用 wcstombs 将 wcs 宽字符串转换成了普通的 mbs，使用 printf 输出没有任何结果，这是为什么呢。

**可以试想一下，wprintf 是使用 wide stream，printf 是使用的 byte stream，两种不同的流模式是不能同时使用的，只能显示最先使用的那种。**

当然，如果直接使用 printf 输出 wcs 也是可以的

	printf ("%ls\n", wcs);
	
使用了 %ls，printf 会将对应的参数视为宽字符串(wcs)，而 printf 又对应byte stream，因此这里要对宽字符(wcs)进行转换，变成普通的字符串(mbs)。这里的转换是 printf 通过对每个宽字符隐式的调用 wcrtomb ()这个标准库函数完成的。按么，wcrtomb() 这个函数进行是按照什么规则进行转换的？这就是setlocale()的作用所在了，wcrtomb 会依据程序员设定的 locale，将 wcha_t中存放的码值，转换为相应的的多字节编码。

参看资料： <br>
1. 软件随想录 Joel Spolsky 著，软件开发者不可不知的 Unicode 和字符集知识 <br>
2. [linux 输出中文](http://blog.csdn.net/magicxiaoz/article/details/4347414) <br>
3. [译文UTF-8 and Unicode FAQ for Unix/Linux ](http://blog.csdn.net/lovekatherine/article/details/1765903) <br>
4. [也谈计算机字符编码](http://blog.csdn.net/bigwhite20xx/article/details/1864908) <br>
5. [字符编码笔记：ASCII，Unicode和UTF-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
