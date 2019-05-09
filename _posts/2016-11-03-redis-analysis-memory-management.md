---
layout: post
title: "redis源码分析 -- 内存管理"
date: 2016-11-03 23:30:00
tags: redis 内存管理
---
redis 内存管理

在 `zmalloc.h` 这个头文件中，有 `USE_TCMALLOC` 和 `USE_JEMALLOC` 这两个宏，分别控制 redis 使用的是 tcmalloc 还是 jemalloc 这两个内存管理器。 tcmalloc 是`Google gperftools`里的组件之一。全名是 `thread cache malloc`（线程缓存分配器），其内存管理分为**线程内存**和**中央堆**两部分。而 jemalloc 是由 `FreeBSD` 的开发人员 `Jason Evans` 开发的，在 FreeBSD、NetBSD和 FireFox 中是默认的 malloc，目前是 Maridab 、Tengine、Redis 中默认推荐的内存优化工具。在没有这两个内存管理器的情况下，redis 使用的是 glibc 的 malloc。下面讲解的都是使用 libc 库的源码分析。

	#ifndef ZMALLOC_LIB
	#define ZMALLOC_LIB "libc"
	#endif

redis 对 tcmalloc 和 jemalloc 的函数都进行了封装

	#if defined(USE_TCMALLOC)
	#define malloc(size) tc_malloc(size)
	#define calloc(count,size) tc_calloc(count,size)
	#define realloc(ptr,size) tc_realloc(ptr,size)
	#define free(ptr) tc_free(ptr)
	#elif defined(USE_JEMALLOC)
	#define malloc(size) je_malloc(size)
	#define calloc(count,size) je_calloc(count,size)
	#define realloc(ptr,size) je_realloc(ptr,size)
	#define free(ptr) je_free(ptr)
	#endif
	
在 redis 中定义了下面这三个变量

	static size_t used_memory = 0;
	static int zmalloc_thread_safe = 0;
	pthread_mutex_t used_memory_mutex = PTHREAD_MUTEX_INITIALIZER;
	
`used_memory` 表示系统使用的内存大小，全局维护这么一个变量，说明作者希望通过这个变量来反映内存的使用情况。`zmalloc_thread_safe`通过变量名也能看出，这个是控制线程安全模式的变量，后面的 mutex 变量  `used_memory_mutex` 就是用在线程安全条件下的互斥信号量。 <br>

`HAVE_ATOMIC`定义了原子操作

	#define update_zmalloc_stat_add(__n) __sync_add_and_fetch(&used_memory, (__n))
	#define update_zmalloc_stat_sub(__n) __sync_sub_and_fetch(&used_memory, (__n))
	
gcc 从 `4.1.2` 提供了 `__sync_*` 系列的 `built-in` 函数，用于提供加减和逻辑运算的原子操作。

{% highlight ruby %}
	// 返回更新前的值
	type __sync_fetch_and_add (type *ptr, type value, ...)
	type __sync_fetch_and_sub (type *ptr, type value, ...)
	type __sync_fetch_and_or (type *ptr, type value, ...)
	type __sync_fetch_and_and (type *ptr, type value, ...)
	type __sync_fetch_and_xor (type *ptr, type value, ...)
	type __sync_fetch_and_nand (type *ptr, type value, ...)
	
	// 返回更新后的值
	type __sync_add_and_fetch (type *ptr, type value, ...)
	type __sync_sub_and_fetch (type *ptr, type value, ...)
	type __sync_or_and_fetch (type *ptr, type value, ...)
	type __sync_and_and_fetch (type *ptr, type value, ...)
	type __sync_xor_and_fetch (type *ptr, type value, ...)
	type __sync_nand_and_fetch (type *ptr, type value, ...)
{% endhighlight %}

如果没有定义 `HAVE_ATOMIC`这个宏，使用 mutex 实现对 `used_memory` 的安全操作

	#define update_zmalloc_stat_add(__n) do { \
	    pthread_mutex_lock(&used_memory_mutex); \
	    used_memory += (__n); \
	    pthread_mutex_unlock(&used_memory_mutex); \
	} while(0)
	
	#define update_zmalloc_stat_sub(__n) do { \
	    pthread_mutex_lock(&used_memory_mutex); \
	    used_memory -= (__n); \
	    pthread_mutex_unlock(&used_memory_mutex); \
	} while(0)

## 申请
内存申请 zmalloc 函数，调用的仍然是 glibc 的 malloc 函数。

	void *zmalloc(size_t size) {
	    void *ptr = malloc(size+PREFIX_SIZE);
		
		// 当申请的 ptr 为 NULL 时，调用 oom 函数处理方法
	    if (!ptr) zmalloc_oom_handler(size);
	#ifdef HAVE_MALLOC_SIZE
	    update_zmalloc_stat_alloc(zmalloc_size(ptr));
	    return ptr;
	#else
	    *((size_t*)ptr) = size;		
	    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
	    return (char*)ptr+PREFIX_SIZE;
	#endif
	}
	
当发生 oom(out of memory) 时，调用 `zmalloc_oom_handler(size)` 异常处理方法。宏 `HAVE_MALLOC_SIZE`只有在定义 `USE_TCMALLOC` 或者 `USE_JEMALLOC` 或者 `__APPLE__`中时才会被定义，在 `ZMALLOC_LIB` 即 libc 中没有被定义，所以此处会跳过这里，执行下面那段。

每次申请内存，申请的大小都是 `size+PREFIX_SIZE`， `PREFIX_SIZE` 的定义为

	#define PREFIX_SIZE (sizeof(size_t))
	
zmalloc函数最后

	    *((size_t*)ptr) = size;	
		
将 size 大小保存在 ptr 中，相当于将申请的长度 size 保存在了 ptr 的头部，然后返回 `ptr + PREFIX_SIZE` 后面的可用部分 `realptr`。

`update_zmalloc_stat_alloc(size + PREFIX_SIZE)` 是先将自己内存对齐，如果long是4位就对齐到4的整数倍。然后将内存的大小记录下来保存到全局变量 `used_memory` 中

	#define update_zmalloc_stat_alloc(__n) do { \
	    size_t _n = (__n); \
	    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
	    if (zmalloc_thread_safe) { \
	        update_zmalloc_stat_add(_n); \
	    } else { \
	        used_memory += _n; \
	    } \
	} while(0) 

	#define update_zmalloc_stat_free(__n) do { \
	    size_t _n = (__n); \
	    if (_n&(sizeof(long)-1)) _n += sizeof(long)-(_n&(sizeof(long)-1)); \
	    if (zmalloc_thread_safe) { \
	        update_zmalloc_stat_sub(_n); \
	    } else { \
	        used_memory -= _n; \
	    } \
	} while(0)
	
对上面这个宏函数的解释： <br>

1. `_n&(sizeof(long)-1)`：这个判断条件，是判断如果按照 long 对齐，假如 `sizeof(long)` 是4，那么对齐后的字节大小应该是4的整数倍，也就是说对齐后的字节数的低两位都是0，即 `_n&(sizeof(long)-1) == 0`，否则就没有对齐，需要将 `_n` 加上 `sizeof(long)-(_n&(sizeof(long)-1))` 对齐，成为4的整数倍。
2. 通过 `zmalloc_thread_safe` 判断是否在安全模式下，如果是安全模式，通过 `update_zmalloc_stat_add(_n)` 更新 `used_memory` 的值，否则直接更新 `used_memory`，将系统使用内存大小更新为最新值。

下面介绍另外几个函数

	void *zcalloc(size_t size) {
	    void *ptr = calloc(1, size+PREFIX_SIZE);
	
	    if (!ptr) zmalloc_oom_handler(size);
	#ifdef HAVE_MALLOC_SIZE
	    update_zmalloc_stat_alloc(zmalloc_size(ptr));
	    return ptr;
	#else
	    *((size_t*)ptr) = size;
	    update_zmalloc_stat_alloc(size+PREFIX_SIZE);
	    return (char*)ptr+PREFIX_SIZE;
	#endif
	}
	
`zcalloc` 调用 `calloc`

	void *zrealloc(void *ptr, size_t size) {
	#ifndef HAVE_MALLOC_SIZE
	    void *realptr;
	#endif
	    size_t oldsize;
	    void *newptr;
	
	    if (ptr == NULL) return zmalloc(size);
	#ifdef HAVE_MALLOC_SIZE
	    oldsize = zmalloc_size(ptr);
	    newptr = realloc(ptr,size);
	    if (!newptr) zmalloc_oom_handler(size);
	
	    update_zmalloc_stat_free(oldsize);
	    update_zmalloc_stat_alloc(zmalloc_size(newptr));
	    return newptr;
	#else
	    realptr = (char*)ptr-PREFIX_SIZE;
	    oldsize = *((size_t*)realptr);	// 长度之前是保存在头部的
	    newptr = realloc(realptr,size+PREFIX_SIZE);
	    if (!newptr) zmalloc_oom_handler(size);
	
	    *((size_t*)newptr) = size;
	    update_zmalloc_stat_free(oldsize);		// 调整used_memory 的大小，减去之前的oldsize，加上新的size
	    update_zmalloc_stat_alloc(size);
	    return (char*)newptr+PREFIX_SIZE;
	#endif
	}
	
`zrealloc` 调用 `realloc`
## 释放
封装 `free` 函数

	void zfree(void *ptr) {
	#ifndef HAVE_MALLOC_SIZE
	    void *realptr;
	    size_t oldsize;
	#endif
	
	    if (ptr == NULL) return;
	#ifdef HAVE_MALLOC_SIZE
	    update_zmalloc_stat_free(zmalloc_size(ptr));
	    free(ptr);
	#else
	    realptr = (char*)ptr-PREFIX_SIZE;	// 获取真实的指针首地址
	    oldsize = *((size_t*)realptr);
	    update_zmalloc_stat_free(oldsize+PREFIX_SIZE);	// 调整 used_memory 的值
	    free(realptr);
	#endif
	}

## 获取内存使用大小
获取 `used_memory` 这个全局变量的值，需要考虑到是否是线程安全模式下，如果是线程安全，需要通过互斥量 mutex 来访问。

	size_t zmalloc_used_memory(void) {
	    size_t um;
	
	    if (zmalloc_thread_safe) {
	#if defined(__ATOMIC_RELAXED) || defined(HAVE_ATOMIC)
	        um = update_zmalloc_stat_add(0);
	#else
	        pthread_mutex_lock(&used_memory_mutex);
	        um = used_memory;
	        pthread_mutex_unlock(&used_memory_mutex);
	#endif
	    }
	    else {
	        um = used_memory;
	    }
	
	    return um;
	}

## Redis中的内存分配大小和碎片大小
### RSS (resident set size)
{% highlight ruby %}
	/* Get the RSS information in an OS-specific way.
	 *
	 * WARNING: the function zmalloc_get_rss() is not designed to be fast
	 * and may not be called in the busy loops where Redis tries to release
	 * memory expiring or swapping out objects.
	 *
	 * For this kind of "fast RSS reporting" usages use instead the
	 * function RedisEstimateRSS() that is a much faster (and less precise)
	 * version of the function. */
	
	#if defined(HAVE_PROC_STAT)
	#include <unistd.h>
	#include <sys/types.h>
	#include <sys/stat.h>
	#include <fcntl.h>
	
	size_t zmalloc_get_rss(void) {
	    int page = sysconf(_SC_PAGESIZE);	// 得到内存页大小
	    size_t rss;
	    char buf[4096];
	    char filename[256];
	    int fd, count;
	    char *p, *x;
	
	    snprintf(filename,256,"/proc/%d/stat",getpid());
	    if ((fd = open(filename,O_RDONLY)) == -1) return 0;
	    if (read(fd,buf,4096) <= 0) {
	        close(fd);
	        return 0;
	    }
	    close(fd);
	
	    p = buf;
		/* 在/proc/pid/stat 文件中的第23个值 */
	    count = 23; /* RSS is the 24th field in /proc/<pid>/stat */
	    while(p && count--) {
	        p = strchr(p,' ');
	        if (p) p++;
	    }
	    if (!p) return 0;
	    x = strchr(p,' ');
	    if (!x) return 0;
	    *x = '\0';
	
	    rss = strtoll(p,NULL,10);
	    rss *= page;
	    return rss;
	}
	#elif defined(HAVE_TASKINFO)
	#include <unistd.h>
	#include <stdio.h>
	#include <stdlib.h>
	#include <sys/types.h>
	#include <sys/sysctl.h>
	#include <mach/task.h>
	#include <mach/mach_init.h>
	
	size_t zmalloc_get_rss(void) {
	    task_t task = MACH_PORT_NULL;
	    struct task_basic_info t_info;
	    mach_msg_type_number_t t_info_count = TASK_BASIC_INFO_COUNT;
	
	    if (task_for_pid(current_task(), getpid(), &task) != KERN_SUCCESS)
	        return 0;
	    task_info(task, TASK_BASIC_INFO, (task_info_t)&t_info, &t_info_count);
	
	    return t_info.resident_size;
	}
	#else
	size_t zmalloc_get_rss(void) {
	    /* If we can't get the RSS in an OS-specific way for this system just
	     * return the memory usage we estimated in zmalloc()..
	     *
	     * Fragmentation will appear to be always 1 (no fragmentation)
	     * of course... */
	    return zmalloc_used_memory();
	}
{% endhighlight %}

在 linux 系统中，RSS (resident set size) 为常驻内存，在 `Wikipedia`中的解释如下

> <p style="font-family:Consolas">In computing, resident set size (RSS) is the portion of memory occupied by a process that is held in main memory (RAM). The rest of the occupied memory exists in the swap space or file system, either because some parts of the occupied memory were paged out, or because some parts of the executable were never loaded. </p>

也就是说，进程所占内存，一部分是RSS常驻内存，一部分存在于交换区(swap space)或文件系统，或被交换出去的部分(paged out)，还有一部分因为没有用到，存在于磁盘没有被加载(never loaded)。

　　在现代操作系统中，内存分配一般都是指虚拟内存(virtual memory)。操作系统创建一个进程时，会为进程分配进程空间，即进程的虚拟地址空间(VmSize)，同时会在进程的页表(page table)中创建相应的页表项。操作系统在初始化进程的页表时，并没有用到物理内存 (Pages are initially not associated with phyiscal memory frames)。当分配的这部分内存用于读写时，会产生一个页中断，操作系统会映射一块空闲的物理内存用于读写。此时，进程的常驻内存就会增加(resident set size，VmRSS)。 <br>
　　根据操作系统中的内存页的置换原则（最久未使用原则），当系统中其他进程需要用到内存时，系统会将不常用的页(infrequently used page)置换出去，将这些页的内容存储到持久化的存储介质上，比如磁盘或者交换区。此时，进程(页被交换出去的进程)的常驻内存 (RSS) 将会减小，但是虚拟内存 (VmSize) 保持不变 (intact)。当交换出去的页被再次使用时，又会被系统置换进内存。虚拟内存大小(Virtual Memory Size)只有在分配的虚拟内存被释放时才会较小。 <br>
　　进程内存申请一般可以分为两种，一种是静态申请的内存，俗称静态存储区(statically allocated memory)，另一种是动态申请的内存，堆内存(heap memory)。静态存储区保存的一般是全局或者静态的变量，位于程序的数据段中，可通过 `VmData` 查看。数据段还包含程序动态申请的堆内存，它是连续的，从低地址往高地址部分增加（栈内存刚好相反，从高地址部分开始，往低地址增加）。数据段中的堆内存是通过特殊的堆内存分配器管理的，它能够将数据段细分成更小的内存块。另一方面，在 linux 系统中，还可以通过直接映射虚拟内存的方式动态申请内存，这种方式一般是为了保护内存，申请大的内存块时才使用的，而且内存申请的大小必须是内存页（一般为 4Kb）的整数倍。 <br>
　　栈在内存使用中也非常重要，尤其是在申请大的数组或者自动变量时(auto)，栈一般是从未被使用的虚拟内存空间的地址顶端开始，往下增长（地址逐渐减小）。在某些情况下，如果栈的地址达到了数据段的顶部(数据段中堆是往地址高的方向增长，同时是朝着栈的方向增长，而栈是从高地址往低地址增长，方向相反)或者达到虚拟内存空间的边界时，可能会出现一些异常情况。栈的大小可通过 `VmStack` 或者 `VmSize` 得到。

总之，

* `VmSize` 包含所有的分配的虚拟内存（如映射的文件--可执行文件，动态库文件等），几乎每一次有新的内存被分配它都会增加。只有当分配的内存被释放时，它才会减少。
* `VmRSS` 当内存页被置换到内存时增加，置换出时减少
* `VmData` 在堆内存被利用时会增加，而且当内存被释放时不会减少，这样可在下一次内存重新使用时进行分配。

> 以上内容翻译自 [stackoverflow.com/questions/13308684/]，以本人拙劣的英文功底，不明白之处可直接查看原文，the first answer。

redis中给出了三种方法获取程序的常驻内存RSS，当定义宏 `HAVE_PROC_STAT` 时，程序从 `/proc/pid/stat` 文件中获取，这个文件中的内容是以空格隔开的数字，第24个代表的就是RSS，这个数字单位是以内存页为单位的（每一个内存页大小为 4K）

	int page = sysconf(_SC_PAGESIZE);
	
通过上面这个系统调用获取系统页的大小，在 `man sysconf` 中关于 `_SC_PAGESIZE` 的说明为 `Size  of a page in bytes.  Must not be less than 1.`，然后通过读取 `/proc/pid/stat` 获取 RSS 所占内存页数 rss，返回RSS的大小为 `page * rss`。

如果定义了 `HAVE_TASKINFO`，可以通过 task_info 直接获取程序的RSS，在Unix系统中好像可以。

最后一种方法就是将 `used_memory` 近似的作为RSS。

### 计算内存使用率 (fragmentation ratio)
	/* Fragmentation = RSS / allocated-bytes */
	float zmalloc_get_fragmentation_ratio(size_t rss) {
	    return (float)rss/zmalloc_used_memory();
	}
	
redis 通过公式 `RSS / allocated-bytes` 来计算内存使用率，通过上面介绍的 RSS 的求值方法可知， `RSS` 是大于等于 `allocated-bytes` (也就是全局变量 `used_memory`)的。所以，`ratio`是大于等于1的。

这里有一个问题，程序都是用多少内存就分配多少内存，哪来的内存碎片？其实，当调用malloc的时候，malloc并不是严格按照参数的值来分配内存。比如，程序只请求一个byte的内存，malloc不会就只分配一个byte，通常，基于内存对齐等方面的考虑，malloc会分配4个byte。这样，如果程序中大量请求1byte内存，那么实际使用的是所请求的4倍。malloc进行小内存分配是很浪费的。所以，碎片就在这里产生了。

### 获取 Private_Dirty 的值
下面这个函数是从 `/proc/self/smaps` 这个文件中读取 `Private_Dirty` 这个字段的值。

{% highlight ruby %}
	/* Get the sum of the specified field (converted form kb to bytes) in
	 * /proc/self/smaps. The field must be specified with trailing ":" as it
	 * apperas in the smaps output.
	 *
	 * Example: zmalloc_get_smap_bytes_by_field("Rss:");
	 */
	#if defined(HAVE_PROC_SMAPS)
	size_t zmalloc_get_smap_bytes_by_field(char *field) {
	    char line[1024];
	    size_t bytes = 0;
	    FILE *fp = fopen("/proc/self/smaps","r");
	    int flen = strlen(field);
	
	    if (!fp) return 0;
	    while(fgets(line,sizeof(line),fp) != NULL) {
	        if (strncmp(line,field,flen) == 0) {
	            char *p = strchr(line,'k');
	            if (p) {
	                *p = '\0';
	                bytes += strtol(line+flen,NULL,10) * 1024;	//将 KB 转换成 byte 
	            }
	        }
	    }
	    fclose(fp);
	    return bytes;
	}
	#else
	size_t zmalloc_get_smap_bytes_by_field(char *field) {
	    ((void) field);
	    return 0;
	}
	#endif
	
	size_t zmalloc_get_private_dirty(void) {
	    return zmalloc_get_smap_bytes_by_field("Private_Dirty:");
	}
{% endhighlight %}

`Private_Dirty` 和 `Private_Clean`，进程fork时，开始内存是共享的，即从父进程那里继承的内存空间都是 `Private_Clean`，运行一段时间之后，子进程对继承的内存空间做了修改，这部分内存就不能与父进程共享了，需要多占用，这部分就是 `Private_Dirty`。

在 linux 系统，有几种可以查看 RSS 的方法：

* 查看 `/proc/pid/stat` 文件，获取第24个数，即为程序的RSS的值，单位是内存页(4k) <br>
![cat /proc/pid/stat](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/redis_analysis/proc_pid_stat.png)

* 查看 `/proc/pid/status` 文件 <br>
![cat /proc/pid/status](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/redis_analysis/proc_pid_status.png) <br>
方法一中的图片上，可以看出第24个数为552，转换成kB大小为 552*4 = 2208kB，大小刚好等于上图中的`VmRSS`

**在 redis 的客户端程序中，执行 `info` 指令，即可查询到内存信息的使用情况等信息。**

redis中，在创建对象和释放对象时，使用了__引用计数的技术__，将在后续源码分析中继续讲解。

> 参考文章: <br>
> 1. https://en.wikipedia.org/wiki/Resident_set_size <br>
> 2. the first answer to stackoverflow.com/questions/13308684/ <br>
> 3. http://blog.csdn.net/jollyjumper/article/details/9100757
