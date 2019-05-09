---
layout: post
title: 如何构建自己的运行库
date: 2019-02-22 17:30:00
tags: 编译 glibc 链接 装载 运行库

---

glibc 提供了运行库，提供了入口函数，下面我们自己来实现一个mini运行库。

# 如何构建自己的运行库

之前介绍了[《main函数之前后》](http://blog.wuzhenyu.com.cn/2019/02/21/what-happened-before-and-after-main.html)，这次，我们试图来构建一个自己的运行库。

> 本篇文章中的例子，来自于俞甲子、石凡、潘爱民的《程序员的自我修养-链接、装载与库》，对例子进行了更新，原书中是32位，这里是64位。也感谢这本书给我带来的帮助，对编译过程和程序的底层知识有了深一层的认识，而这篇文章也作为我的一个笔记和学习成果吧。

[**本文中的源代码地址**](https://github.com/small-cat/myCode_repository/tree/master/minicrt/c)

废话不多说，我们实现的这个运行库，也叫作 minicrt。之前说过，main函数之前，入口函数需要完成各种初始化和准备工作，然后调用main主体函数，main函数结束时，在调用exit负责后续的清理工作。那么先确定我们这个minicrt的基本功能：

- 具有自己的入口函数 `mini_crt_entry`

- 基本的进程退出相关操作 exit

- 支持堆操作 malloc、free

- 支持基本的文件操作 fopen, fwrite. fclose, fread, fseek

- 支持基本的字符串操作 strcpy, strcmp, strlen

- 支持基本的字符串格式化和输出操作 printf sprintf

- 支持 atexit() 函数

简单起见，所有的申明都放在同一个头文件中 minicrt.h

## 入口函数

入口函数名为`mini_crt_entry`，没有参数，也没有返回值，因为exit函数调用的时候，如果正常，函数会直接退出，不会回到入口函数继续执行并返回结果。同时，函数体内还需准备好程序运行的环境，包括main函数的命令行参数，初始化运行库，如堆、I/O等，结束部分主要负责清理程序运行资源。

main函数的两个参数为 argc， argv，argc是参数个数，argv是一个字符串数组，保存的是所有的命令参数。当进程被初始化时，它的堆栈中就保存着环境变量和传递给main函数的参数。汇编指令中，一般函数栈的开头都如下

```assembly
push   %rbp
mov    %rsp,%rbp
sub    $0x20,%rsp
```

这样，将基址寄存器rbp保存下来，然后开辟了一个32字节的栈空间作为函数栈空间。所以说，栈顶寄存器 rsp 指向的位置，是即将初始化的栈空间的顶部，即 rbp 指向的位置。如果我们像下面这样执行函数

```
mini_crt hello
```

命令行参数就是两个，`mini_crt` 和 `hello`，在栈空间初始化之前分布如下所示

```
...
rsp
2
argv[0]'s addr
argv[1]'s addr
...
地址从上往下是递增的，因为栈是往地址小的方向增长的
```

栈空间初始化之后，`push rbp`，然后`mov rsp, rbp`，此时，rbp 的值就成了之前的 rsp，也就是说，rbp+8的值就是2，rbp+16的值就是argv的首地址了（我的环境是64位elementary os）。

完成了获取命令行参数的代码后，还需要在入口函数体内实现对堆和 I/O 的初始化，分别申明为 `mini_crt_heap_init`和 `mini_crt_io_init`。然后调用main主体函数，main函数返回时，调用exit函数退出。exit函数完成两个任务，一个是调用由 atexit() 函数注册的退出回调函数，另一个就是结束进程。入口函数代码如下

```c
void mini_crt_entry(void) {
    int ret;
    int argc;
    char** argv;
     
    char* ebp_reg;
    //ebp_reg = %ebp
    asm(
            "mov %%rbp, %0 \t\n"
            :"=r"(ebp_reg)
       );
     
    // 64bit, the size of rbp is 8 bytes.
    argc = *(long *)(ebp_reg + 8);
    argv = (char**)(ebp_reg + 16);
     
    if (!mini_crt_heap_init()) {
        crt_fatal_error("heap initialize failed.");
    }
     
    if (!mini_crt_io_init()) {
        crt_fatal_error("IO initialize failed.");
    }
     
    // call main functions, and deliver the command line args.
    ret = main(argc, argv);
    exit(ret);
}

/* system call number of sys_exit is 60 */
void exit(int exitCode) {
    asm(
            "mov $0x3c, %%rax \n\t"
            "mov %0, %%rdi \n\t"
            "syscall \n\t"
            ::"m"(exitCode)
       );
}
```

exit 函数，使用系统调用退出，64位系统调用，统一使用 `syscall`，不是32位的 `int 0x08h`。64位的系统调用号也与32位不同，`sys_exit`的系统调用号为60，将 rax 寄存器的值设置为 60，rdi 为返回值，syscall 调用系统调用。

## 堆的实现

堆是一块巨大的内存空间，在这部分空间内，程序可以请求一块连续的内存并自由的使用，这块内存在程序主动放弃之前都会一直保持。如果进程的内存管理由操作系统的内核来做，那么就是说，每次程序申请堆空间，操作系统都要调用系统调用分配一块足够大的内存，给用户程序，从用户态切换到内核态，再切换到用户态，这样非常影响程序的性能。比较好的做法就是程序直接想操作系统一次申请一块适当大的空间，然后有程序自己管理这部分空间，当需要申请内存的时候，程序就成这块空间中切分一块，如果释放，就合并到这块空间中。所以，一般管理对空间分配的都是程序的运行库。

linux 提供了两个系统调用，brk/sbrk 和 mmap 来管理堆空间。在运行库中，有两种最基本的方法来管理堆空间的分配，一个是空闲链表法，一个是位图法。

**空闲链表法**，是将堆中各个空闲块按照链表的方式连接起来，链表采用双向链表的方式，当程序申请空间时，从前往后遍历链表，找到一个合适大小的块分配给程序，当释放空间是，将这块不再使用的空间加入到链表中，然后查看前后是否也是空闲块，如果是，将空闲块合并成一块，减少空间碎片化。当然，实际堆管理比这复杂的多，这这是简单说明一下原理。

**位图法**，是将整个空间划分成大量大小相等的块，用户请求内存的时候，分配整数个数的块给用户。第一块成为头Head，其余成为主体Body，未使用的为Free，所有使用两位即可表示一个块的使用情况，使用一个整数数组就能记录块的使用情况。

这里我们采用双链表的方式，来管理堆空间分配。

### 实现

- 采用空闲链表法管理堆分配

- 堆大小固定为 32MB，然后在这 32MB 中进行空间管理。（仅学习demo使用，尽量简单）

- 使用 brk 系统调用获取 32MB 空间

> 注意，由 brk/sbrk 分配的空间，仅仅只是虚拟地址空间，一开始是不会分配物理内存的，只有当进程试图访问某一个地址的时候，操作系统检测到访问异常，然后为被访问地址所在的页分配物理内存页

先确定链表的结构体

```c
typedef struct _heap_header {
    enum {
        HEAP_BLOCK_FREE = 0xABABABAB,   //magic number of free block
        HEAP_BLOCK_USED = 0xCDCDCDCD   //magic number of used block
    }type;
 
    unsigned size;
    struct _heap_header* next;
    struct _heap_header* prev;
} heap_header;
                                                                                                                                                                          
#define ADDR_ADD(a, o) (((char*)(a)) + o)
#define HEADER_SIZE (sizeof(heap_header))
```

结构体type表示块的状态，是否使用，size为块的大小，next 和 prev 表示双链表节点向前和向后的指针。宏函数 `ADDR_ADD(a, o)` 获取结构体的实际使用内存地址。o 表示 `HEADER_SIZE`时，指针往后偏移，跳过结构体节点的头部，后面的空间就是能够供程序直接使用的空间大小。

brk函数通过 `sys_brk`系统调用来实现

```c
static int brk(void* end_data_segment) {
    int ret = 0;
    // Linux brk system call
    // sys_brk system call number: 12
    // rax:12, rdi:end_data_segment
    asm (
            "mov $12, %%rax \n\t"
            "mov %1, %%rdi  \n\t"
            "syscall        \n\t"
            :"=r"(ret)
            :"b"(end_data_segment)
        );
    
    return ret;
} 
 
int mini_crt_heap_init() {
    void* base = NULL;
    heap_header* header = NULL;
 
    // 32MB heap size
    unsigned heap_size = 1024 * 1024 * 32;
    base = (void*)brk(0);
    void* end = ADDR_ADD(base, heap_size);
    end = (void*)brk(end);
    if (!end) {
        return 0;
    }
                                                                                                                                                                          
    header = (heap_header*)base;
    header->size = heap_size;
    header->type = HEAP_BLOCK_FREE;
    header->next = NULL;
    header->prev = NULL;
    list_head = header;
 
    return 1;
}
```

`mini_crt_heap_init` 函数中，通过 brk 函数申请了32MB的空间，同时初始化和加入空闲链表作为第一个链表节点。

```c
void* malloc(unsigned size) {
    heap_header* header;
    if (0 == size) {
        return NULL;
    }
 
    header = list_head;     //global static variable will be initialized at other place.
    while (NULL != header) {
        if (header->type == HEAP_BLOCK_USED) {
            header = header->next;
            continue;
        }
 
        if ((header->size > size + HEADER_SIZE) &&
                (header->size <= size + HEADER_SIZE * 2)) {
            // header is apt 
            header->type = HEAP_BLOCK_USED;
            return ADDR_ADD(header, HEADER_SIZE);
        }
 
        if (header->size > size + HEADER_SIZE * 2) {
            // block is too big, split into two parts.
            heap_header* split_next = (heap_header*)ADDR_ADD(header, size + HEADER_SIZE);
            split_next->prev = header;
            split_next->next = header->next;
            if (header->next) {
                split_next->next->prev = split_next;
            }
            header->next = split_next;
 
            split_next->size = header->size - (size + HEADER_SIZE);
            split_next->type = HEAP_BLOCK_FREE;
            header->size = size + HEADER_SIZE;
            header->type = HEAP_BLOCK_USED;
 
            return ADDR_ADD(header, HEADER_SIZE);
        }                                                                                                                                                                 
        header = header->next;
    }
    return NULL;
}
```

malloc函数，从链表中遍历寻找合适大小的第一个块，如果块太大，就将块分割。返回NULL表示失败。

```c
void free(void* ptr) {
    heap_header* header = (heap_header*)ADDR_ADD(ptr, -HEADER_SIZE);
    if (header->type != HEAP_BLOCK_USED) {
        return;
    }
 
    header->type = HEAP_BLOCK_FREE;
    //merge if prev or next is also free
    if (header->prev!=NULL && header->prev->type==HEAP_BLOCK_FREE) {
        //merge with prev
        header->prev->next = header->next;
        if (header->next != NULL) {
            header->next->prev = header->prev;
        }
        header->prev->size += header->size;
        header = header->prev;
    }
 
    if (header->next!=NULL && header->next->type==HEAP_BLOCK_FREE) {
        // merge with next
        header->size += header->next->size;
        header->next->prev = NULL;
        header->next = header->next->next;
    }
}
```

free 函数不是真的将块释放，仅仅改变块的状态，设置为未使用的状态，同时，如果前后有空闲的块，就一起合并。

## IO 文件操作

IO 就是对文件的操作，仅支持一下功能：

- 实现 fopen、fread、fwrite、fclose 和 fseek 函数
- 不实现 buffer 缓冲机制
- 支持三个标准输入输出 stdin、stdout、stderr
- 使用内嵌汇编实现 open、read、write、close和 seek 系统调用
- fopen 支持 “r”、“w“、”+“和”a”的几种组合，不对文本模式和二进制模式进行区分

```c
/*************************************************************************
	> File Name: stdio.c
	> Author: Jona
	> Mail: mblrwuzy@gmail.com 
	> Created Time: 2019-02-01 10:37:38
 ************************************************************************/

#include "minicrt.h"

int mini_crt_io_init() {
    //TODO
    //this is a very simple version, does not need to initalize
    //reverse
    return 1;
}

/************************************************************************* 
 * * FUNCTION NAME: open
 * * DESCRIPTION: open file and return file descriptor, implement read 
 * function by system call sys_open.
 * system call number: 2
 * * ARGS: 
 * rax - system call number 0x2
 * rdi - pathname
 * rsi - flags
 * rdx - mode
 * * RETURN VALUE: fd - file descriptor
 * * AUTHOR: Jona
 * * CREATE TIME: 2019-02-01 11:03 
*************************************************************************/ 
static int open(const char* pathname, int flags, int mode) {
    int fd = 0;
    asm (
            "mov $2, %%rax \n\t"
            "mov %1, %%rdi \n\t"
            "mov %2, %%rsi \n\t"
            "mov %3, %%rdx \n\t"
            "syscall \n\t"
            :"=r"(fd)
            :"m"(pathname), "m"(flags), "m"(mode)
        );
    return fd;
}

/************************************************************************* 
 * * FUNCTION NAME: read
 * * DESCRIPTION: implement with system call sys_read
 * * ARGS: 
 * rax = 0x0
 * rdi = fd
 * rsi = buffer
 * rdx = size
 * * RETURN VALUE: 
 * * AUTHOR: Jona
 * * CREATE TIME: 2019-02-01 11:13 
*************************************************************************/ 
static int read(int fd, void* buffer, unsigned long size) {
    int ret = 0;
    asm (
            "mov $0, %%rax \n\t"
            "mov %1, %%rdi \n\t"
            "mov %2, %%rsi \n\t"
            "mov %3, %%rdx \n\t"
            "syscall      \n\t"
            :"=r"(ret)
            :"m"(fd), "m"(buffer), "m"(size)
        );

    return ret;
}

/************************************************************************* 
 * * FUNCTION NAME: write
 * * DESCRIPTION: implement with system call sys_write
 * * ARGS: 
 * rax = 0x01
 * rdi = fd
 * rsi = buffer
 * rdx = size
 * * RETURN VALUE: 
 * * AUTHOR: Jona
 * * CREATE TIME: 2019-02-01 11:18 
*************************************************************************/ 
static int write(int fd, const void* buffer, unsigned long size) {
    // 64位寄存器，只能使用64位的变量存储，如果使用size为unsigned，那么
    // mov到寄存器之后，查看的寄存器状态值不是size的大小
    int ret = 0;
    asm(
            "mov $1, %%rax \n\t"
            "mov %1, %%rdi \n\t"
            "mov %2, %%rsi \n\t"
            "mov %3, %%rdx \n\t"
            "syscall      \n\t"
            :"=r"(ret)
            :"m"(fd), "m"(buffer), "m"(size)
       );

    return ret;
}

/************************************************************************* 
 * * FUNCTION NAME: close
 * * DESCRIPTION: implement with system call sys_close
 * * ARGS: 
 * rax = 0x3
 * rdi = fd
 * * RETURN VALUE: 
 * * AUTHOR: Jona
 * * CREATE TIME: 2019-02-01 11:22 
*************************************************************************/ 
static int close(int fd) {
    int ret = 0;
    asm(
            "mov $3, %%rax \n\t"
            "mov %1, %%rdi \n\t"
            "syscall      \n\t"
            :"=r"(ret)
            :"m"(fd)
       );

    return ret;
}

/************************************************************************* 
 * * FUNCTION NAME: seek
 * * DESCRIPTION: implement with system call sys_lseek
 * * ARGS: 
 * rax = 0x8
 * rdi = fd
 * rsi = offset
 * rdx = mode
 * * RETURN VALUE: 
 * * AUTHOR: Jona
 * * CREATE TIME: 2019-02-01 11:24 
*************************************************************************/ 
static int seek(int fd, int offset, int mode) {
    int ret = 0;
    asm(
            "mov $8, %%rax \n\t"
            "mov %1, %%rdi \n\t"
            "mov %2, %%rsi \n\t"
            "mov %3, %%rdx \n\t"
            "syscall      \n\t"
            :"=r"(ret)
            :"m"(fd), "m"(offset), "m"(mode)
       );

    return ret;
}

FILE* fopen(const char* filename, const char* mode) {
    int fd = -1;
    int flags = 0;
    int access = 00700;     // file permissions

    if (strcmp(mode, "w") == 0) {
        flags |= O_WRONLY | O_CREAT | O_TRUNC;
    }

    if (strcmp(mode, "w+") == 0) {
        flags |= O_RDWR | O_CREAT | O_TRUNC;
    }

    if (strcmp(mode, "r") == 0) {
        flags |= O_RDONLY;
    }

    if (strcmp(mode, "r+") == 0) {
        flags |= O_RDWR | O_CREAT;
    }

    if (strcmp(mode, "a+") == 0) {
        flags |= O_RDWR | O_CREAT | O_APPEND;
    }

    fd = open(filename, flags, access);

    return (FILE*)fd;
}

int fread(void* buffer, int size, int count, FILE* stream) {
    return read((int)stream, buffer, size*count);
}

int fwrite(const void* buffer, int size, int count, FILE* stream) {
    return write((int)stream, buffer, size*count);
}

int fclose(FILE* fp) {
    return close((int)fp);
}

int fseek(FILE* fp, int offset, int set) {
    return seek((int)fp, offset, set);
}
```

64位寄存器，就一定使用8字节的类型变量进行存储，不然会出现一些预想不到的结果。

## 字符串操作

字符串操作也是 minicrt 的一部分，实现字符串拷贝、计算字符串长度、比较两个字符串和整数与字符串之间的转换操作。这部分比较简单。在此不做说明，代码直接去文章开头给出的地址github上面看。

## 字符串格式化

字符串格式化输出，就是我们经常使用的 printf 函数了，我们仅支持对整数和字符串的支持。fputc 和 fputs 函数的实现比较简单，使用我们之前实现的IO 文件操作 fwrite 接口来实现

```c
int fputc(int c, FILE* stream) {
    if (fwrite(&c, 1, 1, stream) != 1) {
        return EOF;
    } else {
        return c;
    }
}

int fputs(const char* str, FILE* stream) {
    int len = (int)strlen(str);
    if (fwrite(str, 1, len, stream) != len) {
        return EOF;
    } else {
        return len;
    }
}
```

对于 printf，fprintf，vfprintf 这些具有可变参数的函数实现，就会复杂一点。我们仿照 `stdarg.h`中的宏定义来实现

```c
/*********** printf OPERATIONS ***********/
/* _cdecl is default, and os push function args into stack from right to left.
 * the growth of stack is from high to low, so from left to right in function
 * args, address is from low to high. in va_arg, t is the last fixed argument,
 * plus offset to get all the unfixed arguments.
 * */

#ifdef ENVIRONMENT32
/* alignment property */
#define _AUPBND                 (sizeof(long) - 1)
#define _ADNBND                 (sizeof(long) - 1)

#define _bnd(X, bnd)            ((sizeof(X) + (bnd)) & (~(bnd)))

#define va_list char*
#define va_start(ap, arg)       ((ap) = (((char*)&(arg)) + (_bnd(arg, _AUPBND))))
/* offset of fixed argument is 32bytes, I don't know why */
#define va_arg(ap, t)           (*(t*)(((ap) += (_bnd(t, _AUPBND))) - (_bnd(t, _ADNBND))))
#define va_end(ap) ((va_list)0)
#else
#include <stdarg.h>
#endif
```

**这里有一个很恶心的地方，就是32位和64位不兼容的问题。**32位系统中，函数参数传递使用的是栈，所以可以直接使用上面的宏来实现就可以，但是64位系统使用寄存器来传递参数，va_start 是一个结构体，不是一个简单的宏，具体实现不清楚，所以直接使用 `stdarg.h` <捂脸>

感兴趣可以查看 [linux ABI](https://software.intel.com/pt-br/articles/linux-abi/)

顺便解释一下上面 32 va 相关的宏的实现。在linux 中，gcc 默认的调用方式一般是 `__cdel`，函数参数入栈的原则是从右往左的，栈的增长方向是从大到小，也就是说，在可变参数的函数参数中，最右边的参数最先入栈，最左边的参数最后入栈，最先入栈的参数，地址是最大的，而最后入栈的参数，地址反而是最小的。所以，`va_start(ap, arg)`中，arg 为参数中最后一个固定参数，这个参数后面就是可变参数的地址，这个参数加上一个偏移就是可变参数的地址了。然后根据可变地址的类型，使用`va_arg`获取可变参数的值。

```c
va_arg(ap, t)           (*(t*)(((ap) += (_bnd(t, _AUPBND))) - (_bnd(t, _ADNBND))))
```

这个宏的实现中，ap的值+=之后就指向了下一个可变参数，但是可变参数的地址并没有变，所以后面再减去偏移获取前一个可变参数，这个宏计算之后，ap就已经指向了下一个可变参数了。

下面给出代码实现

```c
int vfprintf(FILE* stream, const char* format, va_list arglist) {
    int translating = 0;
    int ret = 0;
    const char* p = NULL;
    for (p=format; *p!='\0'; p++) {
        switch(*p) {
        case '%':
            if (!translating) {
                translating = 1;
            } else {    // %%
                if (fputc('%', stream) < 0) {
                    return EOF;
                }
                translating = 1;
            }
            ret++;
            break;
        case 'd':
            if (translating) {
                char buf[16] = {0};
                translating = 0;
                itoa(va_arg(arglist, int), buf);
                if (fputs(buf, stream) < 0) {
                    return EOF;
                }
                ret += strlen(buf);
            } else {
                if (fputc(*p, stream) < 0) {
                    return EOF;
                } else {    // fputc>0
                    ret++;
                }
            }
            break;
        case 's':
            if (translating) { // %s
                const char* str = va_arg(arglist, const char*);
                translating = 0;
                if (fputs(str, stream) < 0) {
                    return EOF;
                }
                ret += strlen(str);
            } else {
                if (fputc(*p, stream) < 0) {
                    return EOF;
                } else {
                    ret++;
                }
            }
            break;
        default:
            if (translating) {
                translating = 0;
            }

            if (fputc(*p, stream) < 0) {
                return EOF;
            } else {
                ret++;
            }
            break;
        }
    }
    return ret;
}

int fprintf(FILE* stream, const char* format, ...) {
    int ret = 0;
    va_list ap;
    va_start(ap, format);
    ret = vfprintf(stream, format, ap);
    va_end(ap);

    return ret;
}

int printf(const char* format, ...) {
    int ret = 0;
    va_list ap;
    va_start(ap, format);
    ret = vfprintf(stdout, format, ap);
    va_end(ap);

    return ret;
}

int fnprintf(int a, char* b, long int c, FILE* stream, const char* format, ...) {
    int ret = 0;
    va_list ap;
    va_start(ap, format);
    ret = vfprintf(stdout, format, ap);
    va_end(ap);

    return ret;
}
```

## minicrt 的使用

至此，minicrt的基本实现已经完成，那么如何编译和使用呢。我的makefile文件如下

```makefile
CC=gcc
CFLAGS=-g -c -fno-builtin -nostdlib -fno-stack-protector
SOURCES=$(shell ls *.c)

STATIC_LIB=minicrt.a

OBJECTS=$(patsubst %.c, %.o, $(SOURCES))

.PHONY:all
all:$(OBJECTS)

.PHONY:clean
clean:
	$(RM) $(OBJECTS) $(STATIC_LIB)

%.o:%.c
	$(CC) $(CFLAGS) -o $@ $<

.PHONY:minicrt
minicrt:
	ar -rs $(STATIC_LIB) malloc.o printf.o stdio.o string.o
```

将目标文件打包成静态库的形式。`-fno-builtin`不让gcc在默认情况使用内部的字符串操作函数，`-nostdlib`表示不使用任何来自 Glibc、Gcc的库文件和启动文件，它包含了`-nostartfiles`这个选项。`-fno-stack-protector`指关闭堆栈保护功能。

## 测试

在当前目录下，创建一个 test 文件夹，然后编写一个简单的测试代码

```c
#include "../minicrt.h"

int main(int argc, char* argv[])
{
    int i;
    int len = 0;
    char buf[256] = {0};
    FILE* fp;
    char** v = malloc(argc * sizeof(char*));
    for (i=0; i<argc; i++) {
        len = strlen(argv[i]);
        v[i] = malloc(len + 2);
        strcpy(v[i], argv[i]);
        v[i][len] = '\n';
        v[i][len+1] = '\0';
    }

    fp = fopen("test.txt", "a+");
    for (i=0; i<argc; i++) {
        len = strlen(v[i]);
        fwrite(v[i], 1, len, fp);
    }
    fclose(fp);

    fp = fopen("test.txt", "r");
    for (i=0; i<argc; i++) {
        len = fread(buf, 1, sizeof(buf), fp);
        buf[len] = '\0';
        printf("%s", buf);

        free(v[i]);
    }
    fclose(fp);

    printf("number:%d string:%s", 123, "thanks");
    fprintf(stdout, "%s\n", "use fprintf");

    return 0;
}
```

编译程序

```makefile
TARGET=mini_test
SOURCES=$(shell ls *.c)
OBJECTS=$(patsubst %.c, %.o, $(SOURCES))

CFLAGS= -fno-builtin -nostdlib -fno-stack-protector

$(TARGET):$(OBJECTS)
	ld -static -e mini_crt_entry -o $@ ../entry.o $^ ../minicrt.a

%.o:%.c
	gcc -g -c $(CFLAGS) -o $@ $^

.PHOMY:clean
clean:
	$(RM) $(OBJECTS) $(TARGET) core
```

程序运行结果如下

```shell
$ ./mini_test 1 2 3 4 5
./mini_test
1
2
3
4
5
use fprintf
use fnprintf
$ cat test.txt
./mini_test
1
2
3
4
5
```

我们再来看下程序的入口函数

```
	...
	20: 0000000000400cfa    51 FUNC    LOCAL  DEFAULT    1 read
    21: 0000000000400d2d    51 FUNC    LOCAL  DEFAULT    1 write
    22: 0000000000400d60    35 FUNC    LOCAL  DEFAULT    1 close
    23: 0000000000400d83    49 FUNC    LOCAL  DEFAULT    1 seek
    24: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS string.c
    25: 00000000004010cb    68 FUNC    GLOBAL DEFAULT    1 strcpy
    26: 0000000000400b4c   185 FUNC    GLOBAL DEFAULT    1 printf
    27: 00000000004006e6   163 FUNC    GLOBAL DEFAULT    1 mini_crt_heap_init
    28: 0000000000400567   339 FUNC    GLOBAL DEFAULT    1 malloc
    29: 0000000000400f25   260 FUNC    GLOBAL DEFAULT    1 itoa
    30: 0000000000400117   122 FUNC    GLOBAL DEFAULT    1 mini_crt_entry
    31: 0000000000400a8e   190 FUNC    GLOBAL DEFAULT    1 fprintf
    32: 0000000000400efc    41 FUNC    GLOBAL DEFAULT    1 fseek
    33: 0000000000400e77    54 FUNC    GLOBAL DEFAULT    1 fread
    34: 0000000000400cbd    11 FUNC    GLOBAL DEFAULT    1 mini_crt_io_init
    ...
```

`mini_crt_entry` 的地址为 0000000000400117，使用 `readelf -h mini_test` 看下入口地址

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x400117
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13976 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         3
  Size of section headers:           64 (bytes)
  Number of section headers:         14
  Section header string table index: 11
```

说明`mini_crt_entry` 就是程序的入口函数。这个例子很简单，还可以继续补充。

**参考：**

1. 程序员的自我修养-链接、装载与库
2. [Linux System Call for X86 64](blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
