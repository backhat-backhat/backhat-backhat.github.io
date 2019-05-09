---
layout: post
title: "代码风格整理"
date: 2017-08-20 23:30:00
tags: 代码风格 约束
---
代码风格整理

## 格式
### 空格
每一行的末尾不应该有空格，不论这行有没有内容。 每个函数声明的后面也不应该有空行。
	
	public string getText () {
	    string text = search_entry.getText ();
	    return text;
	}

每个开始括号之前都应该有一个空格：

	public string getText () {}
	if (a == 5) {
		return 4;
	}
	for (i=0; i<maximum; i++) {}	//空格不要太多
	myFunctionName ();
	Object my_instance = new Object ();

数学表达式中的操作数与运算符之间的空格也是必要的。

	c = n * 2 + 4;

在变量申明初始化时，也需要通过空格对齐，对齐原则为**数据类型 + N个TAB + 变量名 + N个TAB + = + 初始化值**

	int		sutdent_num	= 100;
	unsigned long max_student_num	= 10000;
	char*		student_name	= NULL;

当函数调用或者操作符作为另一个函数的参数时，可以在小括号前不加空格，如下

	char* ptr = (char*) malloc (sizeof(char) * STR_LEN);

### 缩进
使用4个空格的缩进来保持一致性和可读性。在 `Vim` 中可做配置

	tabstop=4
	softtabstop=4
	shiftwidth=4

在类、函数、循环和一般的顺序结构中，左大括号跟在第一行的末尾，接着一行是带了缩进的其他代码，右大括号则应该在单独的一行来关闭函数。

	public int myFunction (int a, string b, long c, int d, int e) {
	    if (a == 5) {
	        b = 3;
	        c += 2;
	        return d;
	    }
	
	    return e;
	}

在条件语句和循环语句当中，**应当总是用大括号包裹住其所属代码，即使只有一行代码**。

	if (my_var > 2) {
	    print ("hello\n");
	}

同样的规则适用于 `else` 和 `else if` ：

	if (a == 4) {
	    b = 1;
	    print ("Yay");
	} else if (a == 3) {
	    b = 3;
	    print ("Not so good");
	}

**如果一个变量的值需要判断两次以上，使用 switch/case 语句替代多个 else/if 。** `if/else` 最多两层嵌套。

	switch (week_day) {
	   case "Monday":
	       message ("Let's work!");
	       break;
	   case "Tuesday":
	   case "Wednesday":
	       message ("What about watching a movie?");
	       break;
	   default:
	       message ("You don't have any recommendation.");
	       break;
	}

### 行长度
每一行字符数不超过80。**但是**，下面这些情况可以超过：<br>

* 如果一行注释包含了超过80字符的命令或 URL，出于复制粘贴的方便可以超过 80 字符；
* 包含长路径的可以超出 80 列，尽量避免；
* 头文件保护可以无视该原则。 <br>

头文件保护，一般是在文件第一行，使用宏保护的方式

	#ifndef HASHMAP_HEADER
	#define HASHMAP_HEADER
	...
	#endif

### 非 ASCII 字符
尽量不使用非 ASCII 字符，使用时必须使用 `UTF-8` 格式。 <br>
哪怕是英文，也不应将用户界面的文本硬编码到源代码中，因此非 `ASCII` 字符要少用。特殊情况下可以适当包含此类字符，如，代码分析外部数据文件时，可以适当硬编码数据文件中作为分隔符的非 `ASCII` 字符串；更常用的是（不需要本地化的）单元测试代码可能包含非 `ASCII` 字符串。此类情况下，应使用 `UTF-8` 格式，因为很多工具都可以理解和处理其编码，十六进制编码也可以，尤其是在增强可读性的情况下——如 "\xEF\xBB\xBF" 是 Unicode 的 `zero-width no-break space` 字符，以 `UTF-8` 格式包含在源文件中是不可见的。

有时候，使用一个新的 terminal 的时候，编码使用默认编码(default)，当代码中有中文注释的时候，打开查看的时候很可能出现乱码的情况，最好在编辑源文件的时候，就将编码设置为 `utf-8` 编码，使用其他工具打开时，通过 `utf-8`
打开就没什么问题。

我的 `Vim` 编码配置为

	"file encoding
	set fencs=utf-8,ucs-bom,shift-jis,gb18030,gbk,gb2312,cp936
	set termencoding=utf-8
	set encoding=utf-8
	set fileencodings=ucs-bom,utf-8,cp936
	set fileencoding=utf-8

如果对编码很感兴趣的童鞋，可以阅读这篇文章 **[http://blog.csdn.net/honglicu123/article/details/53932524]**

### 函数申明、定义或调用的格式
尽量在同一行，如果一行放不下，可以放在多行

	int ret = getMax (a, b);
	void goSomeThing (arg1, arg2, arg3,
			  arg4, arg5) {
		/* statements */
	}

如果参数很多，处于可读性考虑每行可只放一个参数。

	bool retval = DoSomeThing (argument1,
				   argument2,
				   argument3);

如果函数名太长，以至于超过最大长度，可以将所有参数独立成行。

### 布尔表达式
如果一个布尔表达式超过标准行宽(80字符)，需要断行。断行时，每一个子表达式都需要用括号括起来，同时在新的一行的子表达式的开头使用 &&，||符号连接

	if ((this_one_thing > this_other_thing)
	    && (a_third_thing == a_fourth_thing)
	    && (yet_another & last_one)) {
	  ...
	}

**在每一行的开头使用 && 或者 || 将子表达式连接起来，是为了当需要注释其中一个子表达式时，通过 `/**/` 直接注释即可**
## 类和文件
每个类只有一个与其对应的文件。

在所有的文件当中，同一个类的名称应当是一致的。

尽量提取代码中相同的部分，抽象成类以易于扩展。

## 注释
注释都要在同一行内，即使那一行有代码。

在同一行中，代码后面的注释要缩进。过于显眼的注释危害大于好处。

	/* User chose number five */
	if (a == 5) {
	    B = 4;           // Update value of b
	    c = 0;           // No need for c to be positive
	    l = n * 2 + 4;   // Clear l variable
	}

有些人或者公司会硬性的规定注释必须使用英文，推荐使用英文，但是如果英语表达能力有欠缺，分分钟说能够说清楚的，用英文越说越乱，那还是看着办吧(->_->)

### 注释风格
统一使用 `/**/`。

### 变量注释
变量注释，尽量写在变量申明的同一行

	int student_number;		/* student number */

对结构体或者类的成员也这么注释

	typedef struct NodeFileDescription {
		ElemType* date;				/* 值 */
		struct NodeFileDescription* next;	/* 链表指针 */
	};

### 函数注释
在编写函数之前，需要为函数添加相应的注释，函数名称，函数功能，参数说明，作者，邮箱，创建时间，修改时间，如下所示

	/*************************************************************
	 * funcName  :
	 * funcDesc  :
	 * argsDesc  :
	 * author    :
	 * email     :
	 * createTime: .strftime("%Y-%m-%d %H:%M")
	 * modifyTime:
	 * ***********************************************************/

我使用的是 `Vim` 编辑器，所以可以通过键盘映射，使用 `vimscript` 编写这些功能

	:nnoremap <C-m> o/*****************************************************<CR>
				\ * funcName  :<CR>
				\* funcDesc  :<CR>
				\* argsDesc  :<CR>
				\* author    :<CR>
				\* email     :<CR>
				\* createTime:<CR>
				\* modifyTime:<CR>
				\*******************************************************/<ESC>

#### 类的注释
每个类的定义要附着描述类的功能和用法的注释。

### 文件注释
在每一个文件开头加入版权公告，然后是文件内容描述。

可查看 redis 中关于 `ziplist.c` 这个文件的描述，在文件开始，描述了 `ziplist` 的结构，以及 `ziplist entry` 的结构。

最后面一般是一些法律和公告信息，还有作者。如果你对其他人创建的文件做了重大修改，将你的信息添加到作者信息里，这样当其他人对该文件有疑问时可以知道该联系谁。

### 实现注释
对于实现代码中巧妙的、晦涩的、有趣的、重要的地方加以注释。

### TODO注释
对那些临时的、短期的解决方案，或已经够好但并不完美的代码使用 **TODO** 注释。这样的注释要使用全大写的字符串 **TODO**，后面括号（parentheses）里加上你的名字、邮箱等，还可以加上冒号（colon）：目的是可以根据统一的 **TODO** 格式进行查找：

	/* TODO(kl@gmail.com): Use a "*" here for concatenation operator. */
	/* TODO(Zeke) change this to use relations. */


## 通用命名规则
函数命名、变量命名、文件命名应具有描述性，不要过度缩写，类型和变量应该是名词，函数名可以用 “命令性” 动词。 <br>

尽可能给出描述性名称，不要节约空间，让别人很快理解你的代码更重要。

	int num_errors;
	int num_completed_connections; 

### 文件名命名
文件名要全部小写，可以包含下划线（_)。可接受的文件命名：

	my_useful_class.c
	mysql_utils.c

### 类型命名
类型命名每个单词以大写字母开头，不包含下划线：`MyExcitingClass`、`MyExcitingEnum`。 <br>
所有类型命名——**类、结构体、类型定义（typedef）、枚举**——使用相同约定，例如：

	class MyClass { ... }             // Class names
	struct RedisObject { ... } 	// struct name
	typedef unsigned int UINT32;	// typedef
	enum OperatingSystem { ... }// An enum name is the same as

但是，enum 中的成员，可以推荐使用 `enum_` 作为前缀，联合体使用 `uni_` 作为前缀 

### 变量命名
变量名一律小写，单词间以下划线相连，类的成员变量以下划线结尾，如 `my_exciting_local_variable`、`my_exciting_member_variable`。

**普通变量或者局部变量：**

	char* student_name;
	char* studentname;

**作用域前缀**

- m_ : 类成员 members
- ms_ : 类静态成员 static member
- s_ : 静态变量 static
- g_ : 全局变量 global

**全局变量:** <br>
全局变量尽量少用。如果是指针，前缀为 `g_p**`，如果是整型值，使用`g_n**`

	CDataManager* g_pdatamanager;

### 常量命名
在名称前加k：kDaysInAWeek。 <br>
所有编译时常量（无论是局部的、全局的还是类中的）和其他变量保持些许区别，k后接大写字母开头的单词：

	const int kDaysInAWeek = 7;

### 函数命名
以小写字母开头，往后每个单词首字母都需大写。

	void getTablEntry ();
	void geleteUrl ();

**类成员方法命名：**

- 有 1 个或多个单词组成，第 1 个单词首字母小写，往后所有单词首字母大写
- 保护函数 protect，开头加上一个下划线，如 `_setStat ()`
- 私有函数 private，开头加上两个下划线，如 `__destroyImpl ()`
- 虚函数，以 do 开头，如 `doFly ()`
- 接口，以大写字母 `I` 开头

### 枚举类型
枚举属于类型，命名以大小写混合的方式，首字母大写，往后每一个单词首字母均大写。枚举值必须全部大写，单词之间通过下划线连接

	typedef enum UrlTableErrors{
		OK = 0,
		ERROR_OUT_OF_MEMORY,
		ERROR_INVALID_INPUT
	}UrlTableErrors;

### 宏命名
宏名称全部大写，并且多个单词通过下划线连接。 <br>
**注意：**当宏有表达式时，宏定义需要用小括号连接起来

	#define MAX_NUM 100000
	#define ZIPLIST_BYTES(zl)       (*((uint32_t*)(zl)))	// redis中求取压缩链表长度字节数的宏定义

---

为使代码简洁，可读性强，好需要尊重一下一些原则。

1、 使用有意义的变量名和函数名。

- 变量名要能够知名达义。
- 局部变量尽量接近使用它的地方
- 局部变量名字简短，因为按照第二个原则，局部变量很接近使用的地方，能够比较简单的就能获知局部变量的含义
- 不要重用局部变量

2、 函数不要太长，把复杂的逻辑提取出去，做成 “帮助函数” <br>
3、 把复杂的表达式提取出去，做成中间变量，不要将表达式嵌套过深 <br>
4、 在合理的地方换行 <br>
5、 避免使用类成员或者局部变量传递信息，通过返回值的形式传递。 <br>
6、 避免使用自增或者自减表达式，除非是单独的语句或者在 for 循环中 <br>
7、 永远不要省略括号，合理使用括号表示表达式的优先级，默认的运算符的优先级有时候会很让人迷惑，因为需要记住所有运算符的优先级也是一件头疼的事情。

8、 **避免使用 break 和 continue**，解决方法如下：

- 对 continue，可以将 continue 出现的条件反向，消除
- 对 break，可以将 break 出现的条件放在循环体条件中，消除
- 有时可以将 break 换成 return，消除 break
- 如果以上均不适用，可将循环体中复杂的部分提取出来，做成函数调用，以达到去掉 continue 或者 break 的目的

**写直观的代码** <br>
代码接近自然语言描述，不要使用过多的语言特性或者“巧妙”的做法，直接、清晰就好。比如将 if 语句使用 || 或者 && 代替

	conditionA && conditionB || conditionC

条件 A 为 True 时才执行 B，B 为 True 不执行 C，B 为 False 时才执行 C，要理解这个语句，需要多转很多弯，很没必要这么写，直接一点

	if (conditionA) {
		if (conditionB) {
			...
		} else if (conditionC) {
			...
		}
	}

if 语句做多两层嵌套，如果太多，使用 switch 语句代替

**正确处理错误**<br>
1、 尽量考虑和处理每一个可能出现的错误

2、 对每一种 Exception 单独处理，即 catch 异常时，尽量 catch 具体的 Exception，在捕捉到异常时，及时处理，不要继续 throws

**正确处理 NULL 指针** <br>
1、 尽量不要产生 null 指针。不用 NULL 初始化变量，函数不返回 NULL 值。在函数返回结果中，“没有了”或“出错了”都不等同于 NULL，Java 中可使用异常

2、 不要把 NULL 放入 “容器数据结构” 中，即 Array、List、Set 等，也不应该出现在 Map 的 Key 和 Value 中

可以用一个特殊的、真真合法的对象表示“没有了”或者“出错了”，不要用 NULL 表示，不然往后每次都需要对 NULL 进行检查处理。 <br>
**null 不是一个合法的对象，在 lua 中，nil 就是一种数据类型，该数据类型就是 nil，应当这么来看待 NULL，将它作为一种特殊的对象来处理**

3、 函数调用者，要明确理解 NULL 所表示的意义，不同场景下 NULL 表示的意义不同。同时，尽早检查和处理 NULL 返回值，减少 NULL 的传播。不要在检查到 null 的时候，再返回 null

4、 函数作者，明确申明不接受 NULL 作为参数，一旦参数是 NULL，将立即出错，崩溃。

当调用者使用 NULL 作为参数调用函数时，程序崩溃，调用者需要对程序崩溃负全责。

**试图对 NULL 进行容错，就会造成，需要在程序无休止的对 NULL 进行检查和处理，这个让人特别烦。 一开始就对 NULL 零容忍**

**参考文献：**

1. clean code
2. [王垠：编程的智慧](http://blog.jobbole.com/95763/)
