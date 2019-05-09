---
layout: post
title: main 函数之前后
date: 2019-02-21 17:30:00
tags: 编译 glibc 链接 装载

---

初学编程的人都知道，程序是从main函数开始执行的，那么在main函数执行之前和执行之后，程序到底做了些什么呢，比如全局变量是在什么时候执行的呢，我们在动态申请堆内存的时候使用malloc就可以了，那么堆内存是在什么时候初始化的呢？接下来我们就一起探寻答案。

系统在装载程序之后，首先会进行一系列的初始化，为main函数的执行准备好条件，然后再调用main函数，这样就能够执行我们所写的一大堆代码了，执行完后，退出，再进行一些清理工作，大致的步骤如下：

1. 操作系统在创建进程之后，将CPU指令寄存器设置成可执行文件的入口地址，这样，控制权就交到了程序的入口，这就是程序的入口，在Linux中，一般为符号_start，一般为运行库中的某个入口函数
2. 入口函数对运行库和程序运行环境进行初始化，包括堆、I/O、线程、全局变量构造等等
3. 入口函数在完成初始化之后，调用main函数，正式开始执行程序主体部分。
4. main函数执行完毕之后，返回到入口函数，入口函数执行清理工作，包括全局变量析构。堆销毁、关闭I/O等，然后调用系统调用结束进程。

## 编译过程

我们先来看一个非常简单的helloworld.c程序

```c
#include <stdio.h>

int main(int argc, char* argv[]) {
    printf("%s\n", "hello world");
    return 0;
}

```

这段代码输出 hello world 字符串，我们使用 gcc 编译一下，看一下 gcc 的输出信息，`gcc -v helloworld.c`，提取一些关键信息如下

```
/usr/lib/gcc/x86_64-linux-gnu/5/cc1 -quiet -v -imultiarch x86_64-linux-gnu helloworld.c -quiet -dumpbase helloworld.c -mtune=generic -march=x86-64 -auxbase helloworld -version -fstack-protector-strong -Wformat -Wformat-security -o /tmp/ccwmmxIN.s

as -v --64 -o /tmp/cciB2Eo1.o /tmp/ccwmmxIN.s

/usr/lib/gcc/x86_64-linux-gnu/5/collect2 -plugin /usr/lib/gcc/x86_64-linux-gnu/5/liblto_plugin.so -plugin-opt=/usr/lib/gcc/x86_64-linux-gnu/5/lto-wrapper -plugin-opt=-fresolution=/tmp/ccME5V5e.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --sysroot=/ --build-id --eh-frame-hdr -m elf_x86_64 --hash-style=gnu --as-needed -dynamic-linker /lib64/ld-linux-x86-64.so.2 -z relro /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/crt1.o /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/5/crtbegin.o -L/usr/lib/gcc/x86_64-linux-gnu/5 -L/usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/5/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/5/../../.. /tmp/cciB2Eo1.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-linux-gnu/5/crtend.o /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/crtn.o
```

程序编译一共经过了三个步骤。首先调用 cc1 程序，这是gcc的c语言编译器，将 helloworld.c 编译成汇编文件 `/tmp/ccwmmxIN.s`，然后调用 GNU 的汇编程序as，将汇编文件汇编成目标文件 `/tmp/cciB2Eo1.o`，最后，再调用 collect2 程序完成最后的链接，同时收集所有与程序相关的初始化信息并做初始化工作。我们可以看到，在最终生成可执行文件的过程中，collect2 程序将以下目标文件链接进入了可执行文件：

```
crt1.o
crti.o
crtbegin.o
crtend.o
crtn.o
```

这些文件我们先放一下，在后面会讲到。

## 符号分析

我们对可执行文件 a.out 的符号做以下分析

```
readelf -s a.out

	...
    29: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   21 __JCR_LIST__
    30: 0000000000400460     0 FUNC    LOCAL  DEFAULT   14 deregister_tm_clones
    31: 00000000004004a0     0 FUNC    LOCAL  DEFAULT   14 register_tm_clones
    32: 00000000004004e0     0 FUNC    LOCAL  DEFAULT   14 __do_global_dtors_aux
    33: 0000000000601038     1 OBJECT  LOCAL  DEFAULT   26 completed.7594
    34: 0000000000600e18     0 OBJECT  LOCAL  DEFAULT   20 __do_global_dtors_aux_fin
    35: 0000000000400500     0 FUNC    LOCAL  DEFAULT   14 frame_dummy
    36: 0000000000600e10     0 OBJECT  LOCAL  DEFAULT   19 __frame_dummy_init_array_
    37: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS helloworld.c
    38: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    39: 0000000000400708     0 OBJECT  LOCAL  DEFAULT   18 __FRAME_END__
    40: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   21 __JCR_END__
    41: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 
    42: 0000000000600e18     0 NOTYPE  LOCAL  DEFAULT   19 __init_array_end
    43: 0000000000600e28     0 OBJECT  LOCAL  DEFAULT   22 _DYNAMIC
    44: 0000000000600e10     0 NOTYPE  LOCAL  DEFAULT   19 __init_array_start
    45: 00000000004005e0     0 NOTYPE  LOCAL  DEFAULT   17 __GNU_EH_FRAME_HDR
    46: 0000000000601000     0 OBJECT  LOCAL  DEFAULT   24 _GLOBAL_OFFSET_TABLE_
    47: 00000000004005c0     2 FUNC    GLOBAL DEFAULT   14 __libc_csu_fini
    48: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
    49: 0000000000601028     0 NOTYPE  WEAK   DEFAULT   25 data_start
    50: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
    51: 0000000000601038     0 NOTYPE  GLOBAL DEFAULT   25 _edata
    52: 00000000004005c4     0 FUNC    GLOBAL DEFAULT   15 _fini
    53: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
    54: 0000000000601028     0 NOTYPE  GLOBAL DEFAULT   25 __data_start
    55: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    56: 0000000000601030     0 OBJECT  GLOBAL HIDDEN    25 __dso_handle
    57: 00000000004005d0     4 OBJECT  GLOBAL DEFAULT   16 _IO_stdin_used
    58: 0000000000400550   101 FUNC    GLOBAL DEFAULT   14 __libc_csu_init
    59: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT   26 _end
    60: 0000000000400430    42 FUNC    GLOBAL DEFAULT   14 _start
    61: 0000000000601038     0 NOTYPE  GLOBAL DEFAULT   26 __bss_start
    62: 0000000000400526    32 FUNC    GLOBAL DEFAULT   14 main
    63: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
    64: 0000000000601038     0 OBJECT  GLOBAL HIDDEN    25 __TMC_END__
    65: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
    66: 00000000004003c8     0 FUNC    GLOBAL DEFAULT   11 _init
```

在符号中，`_start`符号就是函数的入口，地址为 0x0000000000400430，我们可以通过下面的方法确认

```
readelf -h a.out

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
  Entry point address:               0x400430
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6624 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 28
```

`Entry point address:               0x400430`，这就是入口地址，正好等于`_start`的地址。同时，程序的初始化都是在`_init`中执行的，初始化后，调用main函数，然后，`_fini`负责善后工作。

现在我们来说一下上面链接的时候加入的那些目标文件。正是 crt1.o 文件，提供了程序的入口函数 `_start`，由它负责调用 `__libc_start_main`初始化libc并且调用main函数进入真正的程序主体。

在链接的时候，crt1.o 必须在最开头，然后其他的目标文件都必须在 crti.o 和 crtn.o 之间，才能正确的链接成最终可执行文件。因为在链接过程中，链接器会将目标文件中同名的段合并，只有位于这两个目标文件中间的文件，才能正确的合并。同时，最终合并的 .init 和 .fini 这两个段中的开头和结尾部分，正好来自于 crti.o 和 crtn.o 中的 .init 和 .fini 这两个段。也就是说， .init 和 .fini  这两个段包含的代码形成了完整了的 `_init` 和 `_fini` 函数，分别调用于main函数的前后。

**这些目标文件又都来自于glibc标准库**。glibc的发布版本主要由两部分组成，一部分是头文件，就是我们熟悉的 stdio.h、stdlib.h 等，另一部分则是库的二进制文件部分。而在这些之外，还有几个辅助程序运行的运行库，就是 crt1.o、crti.o、crtn.o 。

## 分析 Glibc 的入口函数

main函数执行之前，首先调用glibc 提供的 `_start` 入口函数，那现在就来分析一下入口函数到底做了一些什么吧。看下`glibc-2.28/sysdeps/x86_64/start.S`的实现如下，提取出关键部分

```assembly
ENTRY (_start)
...
    /* Pass address of our own entry points to .fini and .init.  */
    mov $__libc_csu_fini, %R8_LP
    mov $__libc_csu_init, %RCX_LP
       
    mov $main, %RDI_LP
	call *__libc_start_main@GOTPCREL(%rip)
       
    hlt         /* Crash if somehow `exit' does return.  */
END (_start)
```

代码中，将 `__libc_csu_fini`、`__libc_csu_init`和 `main`作为函数指针，然后传递给 `__libc_start_main`，然后调用这个函数。这个函数在 `csu/libc-start.c`中定义

```c
# define LIBC_START_MAIN __libc_start_main

STATIC int 
LIBC_START_MAIN (int (*main) (int, char **, char ** MAIN_AUXVEC_DECL),
         int argc, char **argv,
#ifdef LIBC_START_MAIN_AUXVEC_ARG
         ElfW(auxv_t) *auxvec,
#endif       
         __typeof (main) init,
         void (*fini) (void),
         void (*rtld_fini) (void), void *stack_end)
```

第一个参数就是main函数指针，argc和argv是命令行参数，这里还包括了环境变量。

init：main函数调用前的初始化工作

fini：main函数结束后的收尾工作

`rtld_fini`：动态加载有关的工作，runtime loader

`stack_end`：栈底的地址

函数体很复杂，我也仅仅看懂了点皮毛，不过可以看到一些很重要的代码。

```c
  char **ev = &argv[argc + 1];
       
  __environ = ev;
       
  /* Store the lowest stack address.  This is done in ld.so if this is
     the code for the DSO.  */
  __libc_stack_end = stack_end;
```

这部分就是提取环境变量，因为命令行参数只有argc个，且以0结束，跳过argc，后面开始，就是从argc+1开始，就是环境变量。同时，保存栈底地址。

```c
/* Note: the fini parameter is ignored here for shared library.  It is registered with __cxa_atexit.  This had the disadvantage that finalizers were called in more than one place.  */

  if (__glibc_likely (rtld_fini != NULL))
    __cxa_atexit ((void (*) (void *)) rtld_fini, NULL, NULL); 
  ...
  if (fini)
    __cxa_atexit ((void (*) (void *)) fini, NULL, NULL);
  ...
  if (init)
    (*init) (argc, argv, __environ MAIN_AUXVEC_PARAM);
```

说明`__cxa_atexit`这个函数是 glibc 的内部函数，这个函数与 atexit 函数作用相同，但是参数不同。上面的语句说明，fini 函数通过 `__cxa_atexit`注册后，会在main函数结束时，被调用。调用 init 函数，进行初始化。这里的init函数，其实是一个函数指针，通过start.S 文件中的函数指针参数，可以看出，这个init函数指针指向的是 `__libc_csu_init`，这个函数定义在 `csu/elf-init.c`文件中。

```
  /* Nothing fancy, just call the function.  */
  result = main (argc, argv, __environ MAIN_AUXVEC_PARAM);
  exit (result); 
```

调用main函数执行主体函数，然后调用exit结束。那么，看下exit函数是如何定义的

```c
void exit (int status)                                                               {
  __run_exit_handlers (status, &, true, true);
}

void
attribute_hidden
__run_exit_handlers (int status, struct exit_function_list **listp,
             bool run_list_atexit, bool run_dtors)
{
  ...
  struct exit_function_list *cur;
  ...
  cur = *listp;
  while (cur->idx > 0)
    {
      struct exit_function *const f = &cur->fns[--cur->idx];
      const uint64_t new_exitfn_called = __new_exitfn_called;
    
      /* Unlock the list while we call a foreign function.  */
      __libc_lock_unlock (__exit_funcs_lock);
      switch (f->flavor)
        {
          void (*atfct) (void);
          void (*onfct) (int status, void *arg);
          void (*cxafct) (void *arg, int status);
    
        case ef_free:
        case ef_us:
          break;
        case ef_on:
          onfct = f->func.on.fn;
#ifdef PTR_DEMANGLE
          PTR_DEMANGLE (onfct);
#endif
          onfct (status, f->func.on.arg);
          break;
        case ef_at:
          atfct = f->func.at;
#ifdef PTR_DEMANGLE
          PTR_DEMANGLE (atfct);
#endif
          atfct ();
          break;                                                                      		  case ef_cxa:
          /* To avoid dlclose/exit race calling cxafct twice (BZ 22180),
         we must mark this function as ef_free.  */
          f->flavor = ef_free;
          cxafct = f->func.cxa.fn;
          #ifdef PTR_DEMANGLE
          PTR_DEMANGLE (cxafct);
#endif
          cxafct (f->func.cxa.arg, status);
          break;
        }
   }
   ...
      *listp = cur->next;
 }
 ...
 _exit(status);
}
```

`exit_function_list`是由 `atexit`、`on_exit`、`__cxa_atexit`注册的函数组成的链表，exit 函数中，遍历链表，同时调用每一个链表中的函数，进行清理工作。链表中函数调用的顺序，是按照先入后出的顺序，即FILO。

## 总结

程序在被操作系统加载后，首先接管控制权，从入口函数处开始执行 _start，然后调用 `__libc_start_main`函数，设置环境变量，栈底地址，注册fini函数，调用init函数初始化，再调用main主体函数，最后调用exit函数，遍历`exit_function_list`执行清理工作，通过 `__exit`系统调用退出程序。

后续会自己实现一个标准库demo，不依赖于glibc，使用 `-fno-builtin -nostdlib` 编译选项编译。

**参考**:

1. 程序员的自我修养 - 链接、装载与库
2. [Linux System Call Table for x86 64](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
