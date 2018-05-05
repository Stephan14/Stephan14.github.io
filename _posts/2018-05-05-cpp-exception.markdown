---
layout:     post
title:      "C++构造函数与析构函数"
subtitle:   "异常处理的问题以及解决方案"
date:       2018-05-05 23:34:00
author:     "邹盛富"
header-img: "img/exception.jpg"
tags:
    - C/C++
---

### 背景
最近写代码时想在构造函数中使用异常处理，在语法上来说，构造函数和析构函数都可以抛出异常。但是，在看别人的代码的时候，发现少有人这么使用，然后就好奇的查了一下，下面的内容都是参考多个博客总结出来的。

### 构造函数

如果在构造函数中抛出异常可能导致一下两种结果：
1. 析构函数不被执行
2. 内存泄露（可以理解为由于析构函数不执行导致的）

#### 析构函数不被执行

**C++仅能 delete 被完全构造的对象，只有一个对象的构造函数完全运行完毕，这个对象才被完全地构造。**所以如果在构造函数中抛出一个异常，这个异常将传递到创建对象的地方（导致程序控制权也会随之转移），这样对象就只是部分被构造，它的析构函数将不会被执行。

有如下测试代码：
```
#include <iostream>
#include <string>
using namespace std;

/******************类定义**********************/
class person {
public:
	person(const string& str):name(str)
	{
		//throw exception();
		cout << "构造一个对象！" << endl;
	};
	~person()
	{
		cout << "销毁一个对象！" << endl;
	};
private:
	string name;
};

/******************测试类**********************/
int main()
{
	try
	{
		person me("songlee");
	}
	catch(exception e)
	{
		cout << e.what() << endl;
	};
	getchar();
	return 0;
}
```
运行上述代码，输出结果如下：
```
构造一个对象！
销毁一个对象！
```
如果在构造函数中抛出一个异常（去掉注释），输出结果如下：
```
测试：在构造函数中抛出一个异常
```
可以看出，析构函数没有被自动执行。为什么“构造一个对象！”也没有输出呢？因为程序控制权转移了，所以在异常点以后的语句都不会被执行。

#### 内存泄露
对于内存泄漏的例子，代码如下：
```
#include <string>
#include <iostream>
using namespace std;

class A {
public:
	A(){};
};

class B {
public:
	B() {
		//throw exception();
		cout << "构造 B 对象!" << endl;
	};
	~B(){ cout << "销毁 B 对象!" << endl; };
};

class Tester {
public:
	Tester(const string& name, const string& address);
	~Tester();
private:
	string theName;
	string theAddress;
	A *a;
	B *b;
};
```
上面声明了三个类（A、B、Tester）,其中Tester类的构造函数和析构函数定义如下：
```
Tester::Tester(const string& name, const string& address):
	theName(name),
	theAddress(address)
{
	a = new A();
	b = new B();  // <——
	cout << "构造 Tester 对象!" << endl;
}

Tester::~Tester()
{
	delete a;
	delete b;
	cout << "销毁 Tester 对象!" << endl;
}
```

在构造函数中，动态的分配了内存空间给a、b两个指针。析构函数负责删除这些指针，确保Tester对象不会发生内存泄露（**C++中delete一个空指针也是安全的**）。

```
int main()
{
	Tester *tes = NULL;
	try
	{
		tes = new Tester("songlee","201");
	}
	catch(exception e)
	{
		cout << e.what() << endl;
	};
	delete tes; // 删除NULL指针是安全的
	getchar();
	return 0;
}
```

看上去好像一切良好，在正常情况下确实没有错。但是在有异常的情况下，恐怕就不会良好了。

试想在 Tester 的构造函数执行时，b = new B()抛出了异常：可能是因为operator new不能给B对象分配足够的内存，也可能是因为 B 的构造函数自己抛出了一个异常。不论什么原因，在 Tester 构造函数内抛出异常，这个异常将传递到建立 Tester 对象的地方（程序控制权也会转移）。

在 B 的构造函数里抛出异常（去掉注释）时，程序运行结果如下：

```
测试：在B的构造函数中抛出一个异常

```
不用为 Tester 对象中的**非指针数据**成员操心，因为它们不是new出来的，且在异常抛出之前已经构造完全，所以它们会自动逆序析构

#### 解决内存泄漏办法
因为当对象在构造中抛出异常后C++不负责清除（动态分配）的对象，所以你必须重新设计构造函数以让它们自己清除。常用的方法是捕获所有的异常，然后执行一些清除代码，最后再重新抛出异常让它继续传递。

示例代码如下：
```
Tester::Tester(const string& name, const string& address):
	theName(name),
	theAddress(address),
	a(NULL),   // 初始化为空指针是必须的
	b(NULL)
{
	try
	{
		a = new A();
		b = new B();  
	}
	catch(...)   // 捕获所有异常
	{
		delete a;
		delete b;
		throw;   // 继续传递异常
	}
}
```
另一种更好的方法是使用**智能指针**。

### 析构函数

《more effective c++》中提出两点理由：

1. 如果析构函数抛出异常，则异常点之后的程序不会执行，如果析构函数在异常点之后执行了某些必要的动作比如释放某些资源，则这些动作不会执行，会造成诸如资源泄漏的问题。

2. 通常异常发生时，c++的机制会调用已经构造对象的析构函数来释放资源，此时若析构函数本身也抛出异常，则前一个异常尚未处理，又有新的异常，会造成程序崩溃的问题。

如果某个操作可能会抛出异常，class应提供一个普通函数（而非析构函数），来执行该操作，目的是给客户一个处理错误的机会。如果析构函数中异常非抛不可，那就用try catch来将异常吞下，但这样方法并不好，我们提倡有错早些报出来。代码例子如下：
```
#include <iostream>
#include <string>
using namespace std;

/******************类定义**********************/
class person {
public:
	person(const string& str):name(str)
	{
		//throw exception();
		cout << "构造一个对象！" << endl;
	};
	~person()
	{
                try {
		throw exception();
		cout << "销毁一个对象！" << endl;
                }
                catch (...)
                {
                  cout << "捕获异常" << endl;
                }
	};
private:
	string name;
};

/******************测试类**********************/
int main()
{
	try
	{
		person me("songlee");
	}
	catch(exception e)
	{
		cout << e.what() << endl;
	};
	getchar();
	return 0;
}
```

![C++ 异常 与 ”为什么析构函数不能抛出异常“ 问题](http://www.cnblogs.com/zhyg6516/archive/2011/03/08/1977007.html)
