---
layout: post
title: "对C++虚函数和虚继承的理解"
date: 2017-03-31 21:30:00
tags: c++ 虚函数 虚继承
---
浅析对C++虚函数与虚继承的理解

> 经常能听到“重载”和“覆盖”的概念。那么这两者之间有什么区别呢？

**重载(overload)**，相对于覆盖(override)来说，是水平方向上的，同一个类中存在重载的概念。重载，是通过实现多个函数名相同，函数参数不同(参数类型不同，参数个数不同，或者类型和个数都不相同)的方法来实现的。比如

	class A {
		public:
		void printA(){const char* str};
		void printA(int num);
	};
	
A 类中实现了对方法 `printA` 的重载。__注意，仅仅函数返回值不同不构成函数重载。__

函数重载时，编译器根据函数参数表的不同，对同名函数做修饰，然后这些同名函数就变成了不同的函数。通过查看符号表就能看出 <br>
![overload sign tb](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/virtual_func_inherit/overload.PNG)

这两个函数的调用，在编译期间就已经确定了，是静态的，他们的地址在编译期间就绑定了。所以说，重载与多态无关，这只是C++语言的一种特性。

再说**覆盖(override)**，子类与父类之间才存在覆盖，即子类重新定义父类虚函数的做法。覆盖的函数要求有相同的函数名，参数表和返回值。如下所示

{% highlight ruby %}
	class A{
		public:
		vritual void print() {
			cout << "A:print()"<< endl;
		}
	};

	class B : public A {
		public:
		virtual void print() {
			cout << "B:print()" << endl;
		}
	}
{% endhighlight %}

类 B 继承 类 A，同时对 A 中的虚函数 print() 重新定义，即覆盖。此时，如果将 B 的指针复制给 A，那么调用 print 函数时，系统根据虚函数地址，能够调用 B 中的 print 函数。

覆盖中，子类重新定义了父类中的虚函数之后，父类指针根据赋给它的不同的子类指针，动态的调用属于子类的该函数，这样的调用在编译期间是无法确定的，因为子类的虚函数地址是无法确定的，只能在运行期间才能确定，所以，覆盖与多态有关。

## 虚函数
C++中虚函数的作用主要是实现多态的机制。多态的本质，就是_允许将子类类型的指针赋值给父类类型的指针_。

虚函数的实现要求对象携带额外的信息，这些信息用于在运行时确定该对象应该调用哪一个虚函数。典型情况下，这一信息具有一种被称为 vptr（virtual table pointer，虚函数表指针）的指针的形式。vptr 指向一个被称为 vtbl（virtual table，虚函数表）的函数指针数组，每一个包含虚函数的类都关联到 vtbl。当一个对象调用了虚函数，实际的被调用函数通过下面的步骤确定： <br>
找到对象的 vptr 指向的 vtbl，然后在 vtbl 中寻找合适的函数指针。 <br>

虚拟函数的地址翻译取决于对象的内存地址，而不取决于数据类型(编译器对函数调用的合法性检查取决于数据类型)。如果类定义了虚函数，该类及其派生类就要生成一张虚拟函数表，即vtable。而在类的对象地址空间中存储一个该虚表的入口，即指向虚函数表的指针，这个入口地址是在构造对象时由编译器写入的。所以，由于对象的内存空间包含了虚表入口，编译器能够由这个入口找到恰当的虚函数，这个函数的地址不再由数据类型决定了。故对于一个父类的对象指针，调用虚拟函数，如果给他赋父类对象的指针，那么他就调用父类中的函数，如果给他赋子类对象的指针，他就调用子类中的函数(取决于对象的内存地址)。

这里我们着重看一下虚函数表。C++的编译器应该是保证虚函数表的指针存在于对象实例中最前面的位置（这是为了保证取到虚函数表有最高的性能——如果有多层继承或是多重继承的情况下）。 这意味着我们通过对象实例的地址得到这张虚函数表，然后就可以遍历其中函数指针，并调用相应的函数。 <br>
下面我们来验证一下，比如：<br>

### 子类没有覆盖
{% highlight ruby %}
	class A {
	    public:
	        virtual void aa1()
	        {
	            cout << "A:aa1()" <<endl;
	        }
	
	        virtual void aa2()
	        {
	            cout << "A:aa2()" <<endl;
	        }
	};
	
	class B {
	    public:
	        virtual void bb1()
	        {
	            cout << "B:bb1()" <<endl;
	        }
	
	        virtual void bb2()
	        {
	            cout << "B:bb2()" <<endl;
	        }
	};
	
	class C : public A, public B {
	    public:
	        virtual void cc1()
	        {
	            cout << "C:cc1()" <<endl;
	        }
	
	        virtual void cc2()
	        {
	            cout << "C:cc2()" <<endl;
	        }
	};
{% endhighlight %}
	
假如我们定义三个变量

	A a;
	B b;
	C c;
	
用 GDB 调试器查看这三个对象的结果如下所示 <br>
![object structure](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/virtual_func_inherit/object_struct.PNG)

可以查看，对象 a 只有一个指向虚函数表的指针 _vptr.A，所以 sizeof(a) 是 8(64位系统)，对象 b 与 a 相同。再看 c，c 继承了类 A 和类 B，所以有两个指向虚函数表的指针 `_vptr.A` 和 `_vptr.B`，同时，c 自己的虚函数指针也是保存在 `_vptr.A` 这个指针所指向的虚函数表里面，因为这是最前面的那个指向的虚函数表。

如果我们定义一个函数指针

	typedef void (*Func)();
	
c 所指向的虚函数表的指针在最前面的话，那么

	(long*)&c
	
这个就是保存指向虚函数表指针的那个地址空间

	(long*)((long*)*((long*)&c + 0) + 0);
	
这就相当于是一个二维指针(因为现在有两个虚函数表指针)，将上述这个指针转换成函数指针，直接调用就能访问类中的成员函数，结果类似下图 <br>
![virtual function pointer](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/virtual_func_inherit/virtual_func_pointer.png)

	Func funcp = NULL;
	funcp = (Func)*(long*)((long*)*((long*)&c + 0) + 0);
	funcp ();

	funcp = (Func)*(long*)((long*)*((long*)&c + 0) + 1);
	funcp ();

	funcp = (Func)*(long*)((long*)*((long*)&c + 0) + 2);
	funcp ();

	funcp = (Func)*(long*)((long*)*((long*)&c + 0) + 3);
	funcp ();

	funcp = (Func)*(long*)((long*)*((long*)&c + 1) + 0);
	funcp ();

	funcp = (Func)*(long*)((long*)*((long*)&c + 1) + 1);
	funcp ();
	
这段代码输出结果为

	A:aa1()
	A:aa2()
	C:cc1()
	C:cc2()
	B:bb1()
	B:bb2()
	
可以看出，C++的编译器应该是保证虚函数表的指针存在于对象实例中最前面的位置，同时，若子类有自己的虚函数，那么子类的虚函数出现在最开始的那个虚函数指针所指向的虚函数表中。

### 子类有覆盖
如果我们将上述 class C 改为如下所示

{% highlight ruby %}
	class C : public A, public B {
		public:
		virtual void aa1() {
			cout<<"C:aa1()"<<endl;
		}

		virtual void cc1() {
			cout<<"C:cc1()"<<endl;
		}

		virtual void cc2() {
			cout<<"C:cc2()"<<endl;
		}
	};
{% endhighlight %}
	
此时，c 的结构不变，但是虚函数表中的函数指针将发生了变化，因为发生了覆盖，如下所示 <br>
![virtual function override](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/virtual_func_inherit/virtual_func_override.png)

可知，c 中虚函数表指针指向的虚函数表中，继承自 a 的虚函数 aa1() 被子类 c 覆盖。 <br>
仍用上述函数指针输出，输出结果变为

	C:aa1()
	A:aa2()
	C:cc1()
	C:cc2()
	B:bb1()
	B:bb2()
	
**PS:**任何试图使用父类指针想调用子类中_未覆盖父类的成员函数_的行为，都将别编译器视为非法操作。比如上例中，

	A* a = new C();
	a->cc1();	//error

> 简单继承时，对于每个对象只有一个 vfptr，vfptr 必须初始化为指向某个 VTable，这在构造函数中发生。一旦vfptr被初始化为指向对应的 VTable 的指针，对象就“知道”它自己是什么类型。但是只有虚函数被调用时，这种自我认知才有用。也就是说，正在运行中才知道具体调用的是哪个虚函数。

<font size=4><b>附加问题</b></font>: <br>
为什么构造函数不能为虚函数，但是析构函数可以？ <br>
大家都知道，类中包含虚函数时，该类及其派生类就会生成一个虚函数表，而在构造类对象时，编译器会为每一个该类对象写入一个指向虚函数表的指针 vfptr，也就是虚函数表的入口地址。当调用该虚函数时，通过该入口地址找到虚函数表中该虚函数的地址来调用。如果构造函数为虚函数，在对象未创建时，还没有虚函数表，也没有虚函数表入口地址，找不到构造函数，所以不能将构造函数设置为虚函数。

同时，C++之父 Bjarne Stroustrup也解释过： <br>
<p style="font-family:Consolas">
A
virtual call is a mechanism to get work done given partial
information. In particular, "virtual" allows us to call a
function knowing only an interfaces and not the exact type of the
object. To create an object you need complete information. In
particular, you need to know the exact type of what you want to
create. Consequently, a "call to a constructor" cannot be
virtual.
</p>
也就是说，虚函数时一种虚调用，是一种可以在只有部分信息的情况下工作的机制，特别允许我们调用一个只知道接口，而不知道对象的准确类型的函数。但是构建一个对象，必须要知道准确的对象类型，因此，构造函数不能为虚函数。

参考： <br>
1、 https://www.zhihu.com/question/35632207 <br>
2、 http://blog.sina.com.cn/s/blog_620882f401016ri2.html

## 虚继承
虚继承，是在函数继承时，使用virtual修饰。虚继承时，如果基类有虚函数，子类也有自己的虚函数，那么此时，子类不仅有一个继承自基类的虚函数指针，还有一个自己指向虚函数表的指针，同时还有一个指向虚基类的指针，剩下的就是成员变量了。(__这里需要注意不同编译器的差异__)

> 虚继承的内存布局与编译器相关！！

> g++中，将虚基类内存(包括vfptr)尾加到派生类。如果虚基类有虚函数，派生类一定会有自己的vfptr，如果派生类没有虚函数，该vfptr指向空。

> vs虚继承时，同样将虚基类内存(包括vfptr)尾加到派生类，并且派生类将产生vbptr，其中存放了虚基类对象在派生类对象中的偏移量。派生类对象中有无vfptr与虚基类无关，仅当派生类有虚函数时，才会为其分配vfptr。
如下例子：

{% highlight ruby %}
	class A {
	    char k[3];
	
	    public:
	    virtual void aa(){ cout <<"A:aa()"<<endl;}
	};
	
	class B: public virtual A {
	    char i[3];
	
	    public:
	    virtual void aa() { cout <<"B:aa()"<<endl; }
	    virtual void bb() { cout <<"B:bb()"<<endl; }
	};
	
	class C: public virtual B {
	    char j[3];
	
	    public:
	    virtual void cc(){ cout <<"C:cc()"<<endl; }
	};

	typedef void (*Func)();
	
	int main ()
	{
	    cout << "sizeof (a) = " << sizeof (a)<<endl;
	    cout <<"sizeof (b) = "<<sizeof (b)<<endl;
	    cout <<"sizeof (c) = "<<sizeof (c)<<endl;
	    cout <<"sizeof (d) = "<<sizeof (d)<<endl;
	
	    Func funcp = NULL;
	    funcp = (Func)*((long*)(*((long*)&a)));
	    funcp ();
	
	    funcp = (Func)*((long*)(*((long*)&b)));
	    funcp ();
	
	    funcp = (Func)*((long*)(*((long*)&c)));
	    funcp ();
	
	    funcp = (Func)*((long*)(*(long*)&d));
	    funcp ();

	    return 0;
	}
{% endhighlight %}

输出结果为（64位系统，g++）：

	sizeof (a) = 16
	sizeof (b) = 32
	sizeof (c) = 48
	A:aa()
	B:aa()
	C:cc()
	A:aa()
	D:dd()
	
**分析：** <br>
类 A，有一个成员变量 k[3] 和一个虚函数，编译器会创建一个虚函数表，和指向虚函数表的指针 vfptr_A，64位环境，按照 8 字节对齐，所以 sizeof (A) 的大小为16，结构如下所示

	class A
		- vfptr_A
		- k[3]

类 B，虚继承类 A，同时，有自己的虚函数，所以编译器首先创建一个指向类 B 自己的虚函数表指针，在加上自己的成员变量 i[3] 的大小，在加上 A 的大小(此处会有一个指向 A 的虚函数表指针，但是 g++ 与 VS 编译器不同，没有指向父类的指针)，所以 B 的大小为 8 + 8 + 16 = 32。上述代码中，

	funcp = (Func)*((long*)(*((long*)&b)));
	
这个取到的是 B 的虚函数表指针，执行的是 B 的第一个虚函数。继承自 A 的尾加在 B 后面。

	class B
		- vfptr_B
		- i[3]
		- virtual class A
			- vfptr_A
			- k[3]

同理，可得 C 的大小。

> 这个地方，在看书和看网上资料的时候，被虚函数的内存分布和指向基类的指针给搅拌了好一会，最后发现是编译器的问题。真是囧，还以为自己哪里出错了。

参考： <br>
1. http://blog.csdn.net/a627088424/article/details/47999757 <br>
2. http://blog.csdn.net/haoel/article/details/1948051/ <br>
3. http://blog.csdn.net/hackbuteer1/article/details/7883531
