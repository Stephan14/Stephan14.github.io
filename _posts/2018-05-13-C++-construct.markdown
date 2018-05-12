---
layout:     post
title:      "C++构造函数"
subtitle:   "深入理解拷贝构造函数"
date:       2018-05-13 00:16:00
author:     "邹盛富"
header-img: "img/c6cf4c8fa6830a46c0aaffaa8c15fe82d07e1ae61c0d61b343c49e9ee7452fa0c87ajpg.jpg"
tags:
    - C/C++
---
### 背景
最近研究了一下c++拷贝构造函数，发现了一个以前不知道的密码，首先，我简单写了如下的代码。
```
#include <iostream>
using namespace std;

class A
{
	public:
		A(int i = 0):val(i){
            cout << "construct" << endl;
        }
		A(const A& a):val(a.val){
			cout << "come to copy create" << endl;
		}
	private:
		int val;
};

int main()
{
	A a1 = 3;
	A a2(4);
	return 0;
}
```
使用命令`g++ -o construct construct.cpp`编译这个程序，其运行结果如我所料如下：
```
construct
construct
```
但是当我把代码改成如下的所示的时候，使用上述编译命令的时候却出现了问题。
```
class	A
{
	public:
		A(int i = 0):val(i){
                    cout << "construct" << endl;
        }
    private:
		A(const A& a):val(a.val){
			cout << "come to copy create" << endl;
		}
	private:
		int val;
};

int main()
{
	A a1 = 3;
	A a2(4);
	return 0;
}

```
在编译的时候报出如下的错误：
```
construct.cpp:23:4: error: calling a private constructor of class 'A'
        A a1 = 3;
          ^
construct.cpp:14:3: note: declared private here
                A(const A& a):val(a.val){
```
当我把`A a1 = 3;`这行代码注释掉的时候，发现可以成功编译并运行


### 原因分析

通过上面报出的错误可以看出`A a1 = 3;`调用了`A(const A& a)`函数，难到它不是直接调用`A(int i = 0)`
函数吗？然后上网查了一下，才知道**像这种通过=赋值构造一个对象的时候，首先使用指定构造函数创建一个临时对象，
然后用拷贝构造函数将那个临时对象复制到正在创建的对象**但是C++标准允许编译器在某些特定情况下把复制构造的步骤省略掉，这样就可以节省若干临时变量,
可以用-fno-elide-constructors把这一优化关闭。于是我又使用`g++ -fno-elide-constructors  -o construct construct.cpp`命令将
刚才报错的代码重新编译一下，发现编译过并能够成功运行。

参考

[C++复制构造函数以及赋值操作符](https://my.oschina.net/fergus/blog/124235)
