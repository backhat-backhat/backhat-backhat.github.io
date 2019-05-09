---
layout: post
title: "GDB学习总结"
date: 2017-10-02 23:31:00
tags: GDB Debug

---

GDB 调试学习总结

## 编译程序
当编译程序，使用 gcc/g++ (the GNU c/c++ compiler) 作为编译器时，为了产生调试信息，可以使用 '-g' 的调试选项。老版本的编译器可能会支持 '-gg' 的调试选项，但是这种已经被淘汰，即使你现在使用的编译器版本仍然支持这种编译选项，也不要使用。调试信息都保存在目标文件中。

> Specify '-g' option when you run the complier, to generate the debugging information for your program. This debugging information is stored in the object file; it describes the data type of each variable and function and the correspondence between source line numbers and addresses in excutable code.

## 支持对宏的处理
GDB 能够识别预编译宏定义，并且能够对其展开(show preprocessor macros' expansion)。3.1 版本以后的 GCC 编译器，使用 '-g3' 编译选项和 DWARF 调试格式的情况下，能够提供宏信息。

GDB 能够对含有宏调用的表达式进行求值，能够显示宏展开的结果，能够显示宏定义，包括宏定义的地方。

> GDB can evaluate expressions containing
macro invocations, show the result of macro expansion, and show a macro’s definition, including where it was defined.

### gdb 的宏操作
1、 <br>
macro expand expression <br>
macro exp expression <br>
对宏表达式进行展开

2、 <br>
info macro macroname <br>
显示宏定义，能够显示宏定义的具体位置，比如在源文件的某一行，被哪个文件的哪一行所引用

比如

	#include <stdio.h>
	#include "sample.h"
	#define M 42
	#define ADD(x) (M + x)
	main ()
	{
	#define N 28
	printf ("Hello, world!\n");
	#undef N
	printf ("We’re so creative.\n");
	#define N 1729
	printf ("Goodbye, world!\n");
	}

编译

	gcc -DWARF -g3 sample.c

使用 gdb 进入调试

	(gdb) info macro ADD
	Defined at /home/jimb/gdb/macros/play/sample.c:5
	#define ADD(x) (M + x)
	(gdb) macro expand ADD(1)
	expands to: (42 + 1)

info macro 能够显示宏定义的具体位置，但是有一些是在编译中使用 `-Dname=value` 的形式，来添加宏的，这种宏，GDB在显示它们的位置的时候，显示的源文件的第 0 行，如下所示

	(gdb) info macro __STDC__
	Defined at /home/jimb/gdb/macros/play/sample.c:0
	-D__STDC__=1

> 注意，一般现在版本的 GCC 编译器，可能支持 -gdwarf2 -gdwarf3 -gdwarf4 这几种编译选项，一般推荐使用最新版本的 -DWARF

## 启动程序
进入gdb调试，输入 run 或者 r 运行程序。

## 设置参数
set args 设置程序运行时的命令行参数，比如

	(gdb) set args a b c

在程序运行时，将会传入 a b c 三个参数

show args 显示设置的程序参数，当然，通过 run 命令启动程序时，也可以直接设置参数，如下所示

	run a b c

## 多线程调试 (Debugging Programs With Multiple Threads)
不同操作系统中所说的线程定义可能不同，但是一般来说，总是指进程中的多个线程。线程能够共享进程的地址空间，但是他们又都有个字的私有数据，比如寄存器信息、栈帧等数据。

GDB 为多线程调试提供了一下机制: <br>

- thread thead-id: 线程切换
- info threads: 线程信息查看。会列出所有的线程信息，最前面显示的是线程的编号，然后是线程的 systag，这个是线程在操作系统中的标识，即 LWP id。在最前面含有 * 的线程，表示的是当前线程。调试命令显示的信息都是以当前线程的角度来展示的(Debugging commands show program information from the perspective of the current thread.)
- thead apply [thread-id] [all] args: 通过 thread-id 来指定某个线程执行后面的命令 args，使用 all 指定所有的线程都执行命令 args
- thread-specific breakpoints: 线程特定断点
- set print thread-events: 控制线程开始或结束时打印信息

GDB 支持调试多线程程序，一般多线程调试，支持两种模式，一种是 all-stop 模式，在该模式下，进程中的任何一个线程停止，GDB 都会将进程中的其他线程都停止。另一种模式是 non-stop，这种模式下，当你在调试其中一个线程的时候，其他线程能够自由执行。下面只介绍在 all-stop 模式下的操作。

GDB 即使在锁步(lockstep)的情况下也不能单步执行所有的线程，线程的调度是由操作系统来决定的，而不是有 GDB。当程序中断时，其他的线程可能中断在某个语句的中间而不是在语句的边界处。(Other threads stop in the middle of a statement, rather than at a clean statement boundary, when the program stops.)

在一些操作系统中，可以通过锁住操作系统的调度器，修改 GDB 默认的动作来允许一个单独线程运行。

- set scheduler-locking mode <br>
设置调度器锁定模式。如果关闭 off，所有线程均能运行。如果打开 on，当 inferior 被唤醒的时候只有当前线程能够运行。step 模式优化单步执行。在单步调试时，step 用抢占当前线程的方式来阻止其他线程抢占控制权。当你在 step 单步调试的时候，其他线程将不会运行，除非使用了 `continue`、`until`、`finish`，其他线程才能够自由运行。

除非另一个线程在它自己的时间片里执行到了一个断点，否则他们不会从你正在调试的线程抢夺 GDB 的控制权。

- show scheduler-locking <br>
显示当前调度器锁定模式

## Debugging Forks
在大部分的操作系统中，GDB 不能很好的支持多进程之间的调试。当程序 fork 创建子进程的时候，GDB 仍然继续调试父进程，而子进程能够畅通无阻的继续执行。如果在子进程的代码出设置了断点，那么执行中的子进程，在接收到 SIGTRAP 信号后，就会终止。(The child will get a SIGTRAP signal (unless it catches the signal) will cause it to terminate.)

当然，如果非要调试子进程，也可以通过一种特殊的办法来达到这个目的。就是让子进程 sleep 的方式，通过设置环境变量来 sleep 子进程比在代码中直接 sleep 更加有效，因为当你不需要调试子进程的时候，就不需要 sleep 子进程。当子进程睡眠之后，通过 ps 的方式获取子进程的 pid，然后使用 gdb attach 的方式对子进程进行调试。

现在，在 GNU/Linux 的内核版本2.5.46以后，GDB 已经支持了对多进程的调试。

**set follow-fork-mode mode** <br>
设置调试器对于 fork 和 vfork 的反应。mode 参数可以为 parent 和 child. 

- parent 默认方式，GDB 继续调试父进程，子进程可以顺利执行。
- child 父进程继续执行，调试器调试子进程

**show follow-fork-mode** <br>
展示 GDB 调试器对于 fork 和 vfork 的反应。

在 Linux 中，如果想调试父子进程，使用 `detach-on-fork`

**show detach-on-fork** <br>
**set detach-on-fork mode** 告诉调试器是分离父子进程还是保留对父子进程的调试控制，mode 参数如下所示

- **on** 子进程(或者父进程，这个根据 follow-fork-mode 设置的参数是 child 还是 parent 来决定) 将被分离(detach)，并且能够独立的运行。
- **off** 父子进程都会被 GDB 挂起，其中一个进程能够正常使用 GDB 进行调试，另一个进程将被挂起 (具体是哪一个进程可以调试，取决于 follow-fork-mode 的参数设置)

**如果你设置了 `set detach-on-fork off`， GDB 会保留对所有进程的控制。可以使用 `info inferiors` 来显示所有进程的信息，使用 `inferior infno` 来切换进程。如果想退出对进程的调试，使用 `detach inferiors` 分离当前进程，是进程能够独立运行。`kill inferiors` 能够杀死当前进程**

## Debugging Multiple Inferiors and Programs
GDB 能够在一个会话中运行和调试多个程序，一般大多数情况下，多进程会同时运行着多个线程。

GDB 将运行中的程序状态保存在一个称为 inferior 的对象中，每一个 inferior 对应一个程序，可能在进程之前创建，在进程退出之后仍然保留。每一个 inferior 都有一个唯一的标识符，这个不同于进程的 pid，一般 inferior 都有自己独立的地址空间。查看 inferior 的信息使用 `info inferior`。

- info inferiors 显示所有 GDB 管理的 inferiors 的信息，每一个 inferior 对应不同的进程，显示的信息如下： <br>
a. GDB 分配的 inferior 的编号 <br>
b. 目标系统的 inferior 的标识符 <br>
c. inferior 对应的进程的可执行文件运行中的文件名(可能是绝对路径)

- inferior infno 通过上面得到的编号，在不同的 inferior 之间切换
- add inferior [-copies n] [-exec executable] 添加 n 个 inferior，以 executable 作为可执行文件执行，n 默认为 1，executable 如果为空，那么 inferior 以空开始，不对应具体的进程。
- clone inferior [-copies n] [infno] 添加 n 个与 infno 编号相同的 inferior，n 默认为 1，如果 infno 为空，那么默认为当前 inferior。
- remove-inferiors infno 删除
- detach inferior infno 分离
- kill inferior infno 杀死 inferior

## 为跳转设置书签-检查点 (这个功能还是比较好用的)
在某些系统，比如 GNU/Linix 中，进程的地址空间都是虚拟地址空间，处于安全考虑，这些地址空间是随机确定的。基本上不可能在一个绝对地址上设置一个断点或者观察点，因为随机确定地址，所以每一次运行，符号的地址都会不相同。然而，一个检查点是一个进程的副本，每次重新执行的时候，符号地址都会相同，这样就能避开地址空间随机化的影响。

- checkpoint 保存被调试程序当前执行状态的一个快照。这个命令没有参数，但是每一个检查点都会被分配一个 id，相当于断点 id
- info checkpoints 显示当前调试会话所保存的所有检查点信息 <br>
a. CheckPoint ID <br>
b. Process ID <br>
c. Code Address <br>
d. Source line, or label

- restart checkpoint-id 从 chenckpoint-id 这个检查点出重启程序，变量、寄存器和栈帧信息都恢复到这个检查点时的模样。
- delete checkpoint checkpoint-id 删除检查点

## 将断点信息保存
调试的时候，会不停的对修改好的代码进行调试，如果每次都要重复设置断点，是一件很麻烦和无聊的事情，那么，可以将设置好的断点信息保存下来。

- save breakpoints [filename] 将所有当前的断点定义都保存在文件 filename 中，在后面的调试中，可以直接被读取调用。其实，就是在当前文件夹下保存一个文本文件，记录下所有的断点信息。后面再次调试的时候，使用 `source filename` 的方法就能重新设置所有之前的断点了。

## 检查堆栈 (Examining the Stack)
### 回溯
backtrace <br>
bt <br>
打印每个栈帧的回溯，每帧一行。可以通过系统中断字符，通常为 `Ctrl-c` 在任意时刻终止回溯。

backtrace full <br>
bt full <br>
回溯

大多数程序都有标准的用户入口点–在此处系统库和启动代码转化为用户代码。对于C 是 main 函数。GDB 要是发现回溯里有入口函数，就会结束回溯，避免跟踪到高度系统相关（或通常不关心的）的代码。要是需要检查启动代码，或者限制回溯到层次数量，可以改变其行为：

set backtrace past-main <br>
set backtrace past-main on <br>
回溯会在入口点处继续跟踪下去。

set backtrace past-main off <br>
回溯在入口点处终止。默认行为。

show backtrace past-main <br>
显示当前用户入口点回溯规则。

set backtrace past-entry <br>
set backtrace past-entry on <br>
回溯会在应用程序的内部入口点继续跟踪下去。这个入口点是由连接器在链接程序的时候编码的，很可能是在用户入口点main 函数（或者是相应的）之前被调用。

set backtrace past-entry off <br>
回溯会在程序的内部入口点处终止。默认行为。

show backtrace past-entry <br>
显示当前内部入口点回溯规则。

### 选择堆栈帧
frame n <br>
f n <br>
选择编号为 n 的帧。回忆一下，0 帧是最内层（当前执行）的帧，1 帧是最内层帧所调用的，以此类推。最高编号的帧是 main 函数的。

frame addr <br>
f addr  <br>
选择在地址 addr 上的帧。这个命令在堆栈帧链被 bug 损坏的时候很有用，使得
GDB 可以为所有帧编号。另外，要是程序有多个堆栈的时候，要在它们之间切换就很有用了。

up n <br>
在堆栈里上移n 帧

down n <br>
在堆栈里下移n 帧

### 堆栈帧信息
info frame <br>
info f <br>
这个命令打印选定帧的文本描述，包括：

- 帧地址
- 下一个帧的地址（被这个帧调用的）
- 上一个帧的地址（这个帧的调用者）
- 这个帧对应的源代码的语言
- 帧参数的地址
- 帧的本地变量的地址
- 帧里的程序计数器（调用者帧的执行地址）
- 在帧里保存的计数器

info frame addr <br>
info f addr <br>
打印在地址 addr 上的帧的文本描述，但不选择此帧

info args <br>
打印选定帧的参数，每个参数一行。

info locals <br>
打印选定帧上的本地变量，每个一行。选定帧的执行点上可见的所有变量（声明为静态或自动的均可）。

## 查看数据
### print
print 是最常用的查看数据或者变量的命令，可以简写为 p，后面接参数，可以按照不同格式打印，格式符号如下：

**x** - 将数值作为整型数据，并以16 进制打印 <br>
**d** - 打印带符号整型数据<br>
**u** - 打印以无符号整型数据<br>
**o** - 以8 进制打印整形数据<br>
**t** - 以2 进制打印整形<br>
**a** - 打印地址，打印16 进制的绝对地址和最近符号的偏移量。可以用这个格式找出一个未知地址的位于何处（在哪个函数里）<br>
**c** - 将一个整型以字符常量打印。会打印一个数值和它表示的字符。超出 7 位的ASCII 的数值（大于127）的字符用8 进制的数字替代打印。不用这个格式的话，GDB 将 char，unsigned char 和 signed char 数据作为字符常量打印。单字节的向量成员以整型数据打印。<br>
**f** - 将数据以浮点类型打印。<br>
**s** - 如果可能的话，以字符串形式打印<br>

### 查看内存
使用 x 命令

	x/nfu addr
	x addr

n，f 和 u 都是可选的参数，指定打印多长的内存数据和以何种格式打印之；addr 是需要打印的内存的地址表达式。如果用默认的 nfu，不需要输入反斜杠’/'。有几个命令可以方便的设置 addr 的默认值。 <br>
n，重复次数，10 进制整数；默认是1。指定显示多长的内存（需要和单元长度u 一起计算得到）。 <br>
f，显示格式，显示格式和 print 命令的格式一样（’x',’d',’u',’o',’t',’a',’c',’f',’s’），外加’i' （表示机器指令格式）。默认是’x'（16进制）。默认格式在用x 或print 命令的时候都会改变。 <br>
u，单元大小。单元大小如下：

	b 字节
	h 2 节节
	w 4 字节。默认值。
	g 8 字节。

每次用x 指定单元长度，这个长度就成为默认值，知道下一次用x 再设置。（对于’s’和’i'格式，单元长度会被忽略而不会改写） <br>
addr，要打印的起始位置

### explore
explore arg

arg 既可以是表达式，也可以是一个可见类型。如下代码

	struct SimpleStruct
	{
		int i;
		double d;
	};
	struct ComplexStruct
	{
		struct SimpleStruct *ss_p;
		int arr[10];
	};
	
	struct SimpleStruct ss = { 10, 1.11 };
	struct ComplexStruct cs = { &ss, { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 } };

在 GDB 调试中，使用 explore 命令

	(gdb) explore cs
	The value of ‘cs’ is a struct/class of type ‘struct ComplexStruct’ with
	the following fields:
	ss_p = <Enter 0 to explore this field of type ‘struct SimpleStruct *’>
	arr = <Enter 1 to explore this field of type ‘int [10]’>
	Enter the field number of choice:

可以继续往下选择，这个命令能够看到详细数据类型以及数据类型中的成员的数据。