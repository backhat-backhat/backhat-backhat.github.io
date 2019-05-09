---
layout: post
title: "redis源码分析 - sds字符串结构"
date: 2016-10-27 23:30:00
tags: redis sds
---
redis中简单动态字符串 sds 分析

> 参考书籍： <br>
> Redis设计与实现 (The Design and Implementation of Redis)，黄建宏著。<br>
> 推荐安卓路上的人（Androidlushangderen）的博客专栏: 
> http://blog.csdn.net/column/details/redis-code.html

redis 使用的字符串抽象数据类型为 SDS (simple dynamic string)。其结构如下所示：

    struct sdshdr {
    	unsigned int len;	//表示字符串长度，即字符数组buf已使用的长度
    	unsigned int free;	//表示buf 数组中尚未使用的字节数
    	char buf[];		//保存字符串，这是一个可变数组
    };
	
**上述结构属于C99中的伸缩数组(flexible array)，是对结构体功能的扩展。在结构体的原型申明时，可以申明一个没有指定数组长度的数组，在使用是，通过malloc动态决定结构体变量的数组大小。**

sds 不同于传统的 C 字符串，它的好处显而易见。

1、 能够在常数复杂度内获取字符串的长度，直接返回 len 就可以了。

    static inline size_t sdslen(const sds s) {
    	struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));		//sds s = sdsnew创建的,返回的是
					//struct sdshdr *shnew->buf，往后偏移了8个字节，这里往前偏移8个字节
    	return sh->len;
    }
	
其中的 sds 的定义为

	typedef char* sds;
因为 sdshdr 的 buf 成员是可变长数组， sizeof 是得不到长度的，所以sizeof(sdshdr)为8，每次创建 sdshdr结构体的时候，返回的都是buf这个字符串，即指针位置向后移动了 sizeof (sdshdr)这个长度，所以在上面由字符串获取 sdshdr 结构体的地址的时候，再往前偏移 sizeof (sdshdr) 个地址，然后直接获取长度 len

2、 sds 所有API均是二进制安全的，在 sdsnewlen 函数的注释中，有这么一段注释

     * You can print the string with printf() as there is an implicit \0 at the
     * end of the string. However the string is binary safe and can contain
     * \0 characters in the middle, as the length is stored in the sds header.
	 
sds 字符串的末尾有一个 '\0' 结尾，作为字符串的结尾，但是为了适应各种类型的数据，同时，还是二进制安全的，在buf 的中间是可以有 '\0' 空字符的，因为取数据不是按照字符串的结尾空字符，而是根据 sdshdr 结构体中的 len 长度来获取的。

{% highlight ruby %}
	sds sdsnewlen(const void *init, size_t initlen) {
	    struct sdshdr *sh;
	
	    if (init) {
	        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);	//申请内存
	    } else {
	        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
	    }
	    if (sh == NULL) return NULL;
	    sh->len = initlen;
	    sh->free = 0;	//因为新申请的sds，所以free设置为0，在重新调整时，会增加free的长度
	    if (initlen && init)
	        memcpy(sh->buf, init, initlen);
	    sh->buf[initlen] = '\0';
	    return (char*)sh->buf;	//返回 sds 字符串
	}
{% endhighlight %}

3、 杜绝缓冲区溢出，减少修改字符串带来的内存重复分配的次数 <br>
sds 在调用 sdscat等字符串拼接类的API时，都会调整空间大小，即增加free的大小，保证不会发生缓冲区溢出

{% highlight ruby %}
	//这个函数不会改变字符串的值，也不会改变len 的长度，只是增加了free的大小
	sds sdsMakeRoomFor(sds s, size_t addlen) {
	    struct sdshdr *sh, *newsh;
	    size_t free = sdsavail(s);
	    size_t len, newlen;
	
	    if (free >= addlen) return s;	//当剩余空间充足时，不需要调整
	    len = sdslen(s);
	    sh = (void*) (s-(sizeof(struct sdshdr)));
	    newlen = (len+addlen);
	    if (newlen < SDS_MAX_PREALLOC)	//1024*1024，就是1M
	        newlen *= 2;
	    else
	        newlen += SDS_MAX_PREALLOC;
	    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);
	    if (newsh == NULL) return NULL;
	
	    newsh->free = newlen - len;
	    return newsh->buf;
	}
{% endhighlight %}

调整sds 的空间大小时， 如果执行的是拼接字符串操作，长度为 addlen，那么先判断剩余可分配内存的大小 free，如果空间足够，不需要调整，否则，当 newlen 小于 1M 时，增加newlen 的一倍，拼接后，free 的长度同样为newlen，否则，增加 1M的长度。

而且sds采用的是惰性空间释放策略，并不是真正的释放内存空间

{% highlight ruby %}
	sds sdstrim(sds s, const char *cset) {
	    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
	    char *start, *end, *sp, *ep;
	    size_t len;
	
	    sp = start = s;
	    ep = end = s+sdslen(s)-1;
	    while(sp <= end && strchr(cset, *sp)) sp++;
	    while(ep > start && strchr(cset, *ep)) ep--;
	    len = (sp > ep) ? 0 : ((ep-sp)+1);
	    if (sh->buf != sp) memmove(sh->buf, sp, len);
	    sh->buf[len] = '\0';
	    sh->free = sh->free+(sh->len-len);
	    sh->len = len;
	    return s;
	}
{% endhighlight %}

仅仅是将增加free的长度，并将buf 和len的值进行调整，这样，当下一次对该 sds 进行字符串拼接的操作时，不需要频繁重复的申请空间，减少了内存申请的次数。

sds 具有以下优点：

- 常熟复杂度获取字符串长度
- 杜绝缓冲区溢出
- 减少修改字符串长度时所需的内存重分配次数
- 二进制安全
- 兼容部分C函数。



其他的相关API函数解析

{% highlight ruby %}
	/* Helper for sdscatlonglong() doing the actual number -> string
	 * conversion. 's' must point to a string with room for at least
	 * SDS_LLSTR_SIZE bytes.
	 *
	 * The function returns the length of the null-terminated string
	 * representation stored at 's'. */
	#define SDS_LLSTR_SIZE 21	//long long类型的最大数为 9223372036854775808 = 2^63,而 2^64有20位
	// long long convert to string, return length of string
	int sdsll2str(char *s, long long value) {
	    char *p, aux;
	    unsigned long long v;
	    size_t l;
	
	    /* Generate the string representation, this method produces
	     * an reversed string. */
	    v = (value < 0) ? -value : value;
	    p = s;
	    do {
	        *p++ = '0'+(v%10);	//逐位求余
	        v /= 10;
	    } while(v);
	    if (value < 0) *p++ = '-';
	
	    /* Compute length and add null term. */
	    l = p-s;
	    *p = '\0';
	
	    /* Reverse the string. */
	    p--;
	    while(s < p) {	//倒序，因为long long的最高数字在p的第一个字符位置
	        aux = *s;
	        *s = *p;
	        *p = aux;
	        s++;
	        p--;
	    }
	    return l;	//返回转换后的字符串的长度
	}
{% endhighlight %}

格式化输出函数 sdscatfmt，比sdscatprintf 性能好，因为后者依赖于libc 中的 sprintf() 家族，这种会比较慢，同时直接操作 sds 字符串性能会更好

{% highlight ruby %}
	/* This function is similar to sdscatprintf, but much faster as it does
	 * not rely on sprintf() family functions implemented by the libc that
	 * are often very slow. Moreover directly handling the sds string as
	 * new data is concatenated provides a performance improvement.
	 *
	 * However this function only handles an incompatible subset of printf-alike
	 * format specifiers:
	 *
	 * %s - C String
	 * %S - SDS string
	 * %i - signed int
	 * %I - 64 bit signed integer (long long, int64_t)
	 * %u - unsigned int
	 * %U - 64 bit unsigned integer (unsigned long long, uint64_t)
	 * %% - Verbatim "%" character.
	 */
	sds sdscatfmt(sds s, char const *fmt, ...) {
	    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
	    size_t initlen = sdslen(s);
	    const char *f = fmt;
	    int i;
	    va_list ap;
	
	    va_start(ap,fmt);
	    f = fmt;    /* Next format specifier byte to process. */
	    i = initlen; /* Position of the next byte to write to dest str. */
	    while(*f) {
	        char next, *str;
	        unsigned int l;		//%u
	        long long num;		//%I
	        unsigned long long unum;	//%U
	
	        /* Make sure there is always space for at least 1 char. */
	        if (sh->free == 0) {
	            s = sdsMakeRoomFor(s,1);	//调整free的大小，保证足够的空间
	            sh = (void*) (s-(sizeof(struct sdshdr)));
	        }
	
	        switch(*f) {
	        case '%':
	            next = *(f+1);
	            f++;
	            switch(next) {
	            case 's':
	            case 'S':
	                str = va_arg(ap,char*);
	                l = (next == 's') ? strlen(str) : sdslen(str);
	                if (sh->free < l) {
	                    s = sdsMakeRoomFor(s,l);
	                    sh = (void*) (s-(sizeof(struct sdshdr)));
	                }
	                memcpy(s+i,str,l);		//在当前字符串后面继续append
	                sh->len += l;
	                sh->free -= l;
	                i += l;
	                break;
	            case 'i':
	            case 'I':
	                if (next == 'i')
	                    num = va_arg(ap,int);
	                else
	                    num = va_arg(ap,long long);
	                {
	                    char buf[SDS_LLSTR_SIZE];
	                    l = sdsll2str(buf,num);		//long long convert to string, return sds
	                    if (sh->free < l) {
	                        s = sdsMakeRoomFor(s,l);
	                        sh = (void*) (s-(sizeof(struct sdshdr)));
	                    }
	                    memcpy(s+i,buf,l);
	                    sh->len += l;
	                    sh->free -= l;
	                    i += l;
	                }
	                break;
	            case 'u':
	            case 'U':
	                if (next == 'u')
	                    unum = va_arg(ap,unsigned int);
	                else
	                    unum = va_arg(ap,unsigned long long);
	                {
	                    char buf[SDS_LLSTR_SIZE];
	                    l = sdsull2str(buf,unum);
	                    if (sh->free < l) {
	                        s = sdsMakeRoomFor(s,l);
	                        sh = (void*) (s-(sizeof(struct sdshdr)));
	                    }
	                    memcpy(s+i,buf,l);
	                    sh->len += l;
	                    sh->free -= l;
	                    i += l;
	                }
	                break;
	            default: /* Handle %% and generally %<unknown>. */
	                s[i++] = next;
	                sh->len += 1;
	                sh->free -= 1;
	                break;
	            }
	            break;
	        default:	//不是 % 时，直接将字符放在 s 中
	            s[i++] = *f;
	            sh->len += 1;
	            sh->free -= 1;
	            break;
	        }
	        f++;
	    }
	    va_end(ap);
	
	    /* Add null-term */
	    s[i] = '\0';
	    return s;
	}
{% endhighlight %}

上述这个函数的完整写法可以借鉴，当需要自己处理格式化字符串的时候，libc 提供的格式化字符串的函数 sprintf... 会对性能有一定的影响。

{% highlight ruby %}
	/* Split 's' with separator in 'sep'. An array
	 * of sds strings is returned. *count will be set
	 * by reference to the number of tokens returned.
	 *
	 * On out of memory, zero length string, zero length
	 * separator, NULL is returned.
	 *
	 * Note that 'sep' is able to split a string using
	 * a multi-character separator. For example
	 * sdssplit("foo_-_bar","_-_"); will return two
	 * elements "foo" and "bar".
	 *
	 * This version of the function is binary-safe but
	 * requires length arguments. sdssplit() is just the
	 * same function but for zero-terminated strings.
	 */
	// 函数功能： 将字符串 s 以 sep 分割，并将分割后的结果保存在 token 数组中
	// sep 可以使单个分割字符 ，也可以是一个字符串
	sds *sdssplitlen(const char *s, int len, const char *sep, int seplen, int *count) {
	    int elements = 0, slots = 5, start = 0, j;
	    sds *tokens;
	
	    if (seplen < 1 || len < 0) return NULL;
	
	    tokens = zmalloc(sizeof(sds)*slots);
	    if (tokens == NULL) return NULL;
	
	    if (len == 0) {
	        *count = 0;
	        return tokens;
	    }
	    for (j = 0; j < (len-(seplen-1)); j++) {
	        /* make sure there is room for the next element and the final one */
	        if (slots < elements+2) {	// 默认的token 数组大小为5
	            sds *newtokens;
	
	            slots *= 2;
	            newtokens = zrealloc(tokens,sizeof(sds)*slots);
	            if (newtokens == NULL) goto cleanup;
	            tokens = newtokens;
	        }
	        /* search the separator */
			// 将 s 进行分割，然后保存在 token 中
	        if ((seplen == 1 && *(s+j) == sep[0]) || (memcmp(s+j,sep,seplen) == 0)) {
	            tokens[elements] = sdsnewlen(s+start,j-start);	// j-start 为分割后当前小串的长度
	            if (tokens[elements] == NULL) goto cleanup;
	            elements++;
	            start = j+seplen;	//跳过 sep，到下一个分割的小串的起始位置
	            j = j+seplen-1; /* skip the separator */
	        }
	    }
	    /* Add the final element. We are sure there is room in the tokens array. */
	    tokens[elements] = sdsnewlen(s+start,len-start);
	    if (tokens[elements] == NULL) goto cleanup;
	    elements++;
	    *count = elements;
	    return tokens;
	
	cleanup:
	    {
			// 清空释放 token
	        int i;
	        for (i = 0; i < elements; i++) sdsfree(tokens[i]);
	        zfree(tokens);
	        *count = 0;
	        return NULL;
	    }
	}
{% endhighlight %}