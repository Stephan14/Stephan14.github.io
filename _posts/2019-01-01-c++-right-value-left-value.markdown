---
layout:     post
title:      "C++ 右值的前世今生"
subtitle:   "右值的相关使用方式"
date:       2019-01-01 14:32:30
author:     "邹盛富"
header-img: "img/a657bccde524f45e21195d584d4849a5_hd.jpg"
tags:
    - C++
---

## 问题与现状
现在有如下的代码：
```
#include <stdio.h>                       //1
class B                                  //2
{                                        //3
};                                       //4

B func(const B& rhs){                    //5
    return rhs;                          //6
}                                        //7

int main(int argc, char **argv){         //8
    B b1,b2;                             //9
    b2=func(b1);                         //10
}                                        //11
```

在不考虑编译器优化（NRVO）的前提下，第10行的执行情况如下：
1. 拷贝构造函数：在func函数调用完成之后，返回`B`类型的对象的时候，由于不是返回引用类型，所以会调用拷贝构造函数生成一个临时对象，暂时叫做`tmp`。由于rhs的类型为`const B&`类型，所以此时不会调用析构函数
2. 赋值运算符函数：func函数执行完成之后将上述的`tmp`通过赋值运算符赋值给`b2`
3. `tmp`对象调用相应的析构函数进行析构

通过上述流程的讲解可以看出来，如果没有进行编译器优化的情况下，上述这样的函数会每次都需要构造临时变量并进行析构，为了解决上述引入了右值引用解决上述的问题。

## 左值和右值的定义
- 左值：可被引用的数据对象，例如，变量、数组元素、结构成员、引用和解除引用的指针都是左值。在C语言中，左值最初指的是出现在赋值语句左边的实体，但这是引入const之前的情况。now，常规变量和const变量都可视为左值，因为可通过地址访问它们。常规变量属于可修改的左值，const变量属于不可修改的左值。左值基本上和以前的认知没有太大的改变。
- 右值：字面常量（用括号括起来的字符串除外，因为它们表示地址）、包含多项的表达式以及返回值的函数（条件是该函数返回的不是引用）。

结合上述的定义可以发现，其实`func`函数的返回值就是一个右值。

## 右值的使用
```
void process_value(int& i) {
 std::cout << "LValue processed: " << i << std::endl;
}

void process_value(int&& i) {
 std::cout << "RValue processed: " << i << std::endl;
}

int main() {
 int a = 0;
 process_value(a);
 process_value(1);
}
```
运行结果：
```
LValue processed: 0
RValue processed: 1
```
Process_value 函数被重载，分别接受左值和右值。由输出结果可以看出，临时对象是作为右值处理的。

## 右值的特性

### 允许调用成员函数
有如下的一个例子：
```
class ValueCl
{
public:
	ValueCl(const int &value) :mValue(value) {};

public:
	void SetValue(const int& value) { mValue = value; }
	int GetValue() const { return mValue; }

	ValueCl& operator+(const ValueCl& v)
	{
		mValue += v.mValue;

		return *this;
	}

protected:
	int mValue;
};

int main()
{
	/*
	 * 此处体现了可以在自定义类型的右值对象上调用类方法修改右值属性，
	 * 并且const左值引用可以延长右值的生命周期
	 */
	ValueCl{3}.SetValue(4);

	/*
	 * 对于这种情况下的右值可以出现在赋值表达式的左侧是因为，
	 * ValueCl& operator=(const ValueCl& v);的默认赋值运算符的存在。
	 * 同时可以在右值对象上调用类非const方法
	 */
	ValueCl{1} + ValueCl{2} = ValueCl{ 3 };
}
```
按照上述例子的使用方式，右值是可以被修改的。既然右值可以被修改，那么就可以实现右值引用。

### 只能被 const reference 指向

如果引用参数是const,则编译器将在下面两种情况下生成临时变量：
1. 实参的类型正确，但不是左值
2. 实参的类型不正确，但可以转换为正确的类型

还是文中的第一个例子，我们按照如下的方式修改：

```
void process_value(int& i) {
 std::cout << "LValue processed: " << i << std::endl;
}

void process_value(int&& i) {
 std::cout << "RValue processed: " << i << std::endl;
}

void forward_value(int&& i) {
 process_value(i);
}

int main() {
 int a = 0;
 process_value(a);
 process_value(1);
 forward_value(2);
}
```
运行结果：

```
LValue processed: 0
RValue processed: 1
LValue processed: 2
```
虽然 2 这个立即数在函数 forward_value 接收时是右值，但到了 process_value 接收时，变成了左值。

### 右值不能当成左值使用，但左值可以当成右值使用

## 转移语义

右值引用是用来支持转移语义的，转移语义可以将资源 ( 堆，系统对象等 ) 从一个对象转移到另一个对象，这样能够减少不必要的临时对象的创建、拷贝以及销毁，能够大幅度提高 C++ 应用程序的性能。临时对象的维护 ( 创建和销毁 ) 对性能有严重影响。要实现转移语义，需要定义转移构造函数，还可以定义转移赋值操作符。对于右值的拷贝和赋值会调用转移构造函数和转移赋值操作符。如果转移构造函数和转移拷贝操作符没有定义，那么就遵循现有的机制，拷贝构造函数和赋值操作符会被调用。


### 实现转移构造函数和转移赋值函数

例子：
```
class MyString {
private:
 char* _data;
 size_t   _len;
 void _init_data(const char *s) {
   _data = new char[_len+1];
   memcpy(_data, s, _len);
   _data[_len] = '\0';
 }
public:
 MyString() {
   _data = NULL;
   _len = 0;
 }

 MyString(const char* p) {
   _len = strlen (p);
   _init_data(p);
 }

 MyString(const MyString& str) {
   _len = str._len;
   _init_data(str._data);
   std::cout << "Copy Constructor is called! source: " << str._data << std::endl;
 }

 MyString& operator=(const MyString& str) {
   if (this != &str) {
     _len = str._len;
     _init_data(str._data);
   }
   std::cout << "Copy Assignment is called! source: " << str._data << std::endl;
   return *this;
 }

 virtual ~MyString() {
   if (_data) free(_data);
 }
};

int main() {
 MyString a;
 a = MyString("Hello");
 std::vector<MyString> vec;
 vec.push_back(MyString("World"));
}
```

运行结果：

```
Copy Assignment is called! source: Hello
Copy Constructor is called! source: World
```
MyString(“Hello”) 和 MyString(“World”) 都是临时对象，也就是右值。虽然它们是临时的，但程序仍然调用了拷贝构造和拷贝赋值，造成了没有意义的资源申请和释放的操作。

在上述代码中添加如下：
```
MyString(MyString&& str) {
   std::cout << "Move Constructor is called! source: " << str._data << std::endl;
   _len = str._len;
   _data = str._data;
   str._len = 0;
   str._data = NULL;
}

MyString& operator=(MyString&& str) {
   std::cout << "Move Assignment is called! source: " << str._data << std::endl;
   if (this != &str) {
     _len = str._len;
     _data = str._data;
     str._len = 0;
     str._data = NULL;
   }
   return *this;
}
```
##### 注意
1. 参数（右值）的符号必须是右值引用符号，即“&&”
2. 参数（右值）不可以是常量，因为我们需要修改右值
3. 参数（右值）的资源链接和标记必须修改。否则，右值的析构函数就会释放资源。转移到新对象的资源也就无效了

运行结果：
```
Move Assignment is called! source: Hello
Move Constructor is called! source: World
```

## 标准库函数std::move

既然编译器只对右值引用才能调用转移构造函数和转移赋值函数，而所有命名对象都只能是左值引用，如果已知一个命名对象不再被使用而想对它调用转移构造函数和转移赋值函数，也就是把一个左值引用当做右值引用来使用，怎么做呢？标准库提供了函数 std::move，这个函数以非常简单的方式将左值引用转换为右值引用。

```
void ProcessValue(int& i) {
 std::cout << "LValue processed: " << i << std::endl;
}

void ProcessValue(int&& i) {
 std::cout << "RValue processed: " << i << std::endl;
}

int main() {
 int a = 0;
 ProcessValue(a);
 ProcessValue(std::move(a));
}
```

运行结果 :
```
LValue processed: 0
RValue processed: 0
```

## 完美转发

之所有存在完美转发，其问题实质是：模板参数类型推导在转发过程中无法保证左右值的引用问题。而完美转发就是在不破坏const属性的前提下通过增加左右值引用概念和新增参数推导规则解决这个问题。

可以用如下代码来帮助理解完美转发：
```
template <typename T>
void IamForwarding(T t)
{
    IrunCodeActually(t);
}
```

在上述代码中`IamForwarding`会将参数转发给`IrunCodeActually`函数来执行，但是在函数调用之间还会存在对象的拷贝，因此可以使用引用来解决该问题，但是又要考虑左值引用和右值引用的问题。可以使用重载来解决该问题，但是这样会造成代码的冗余。

最终解决这个问题的方法就是使用引用折叠，在引用折叠中规定：如果两个引用中有一个是左值引用，那么折叠的结果是一个左值引用。否则（即两个都是右值引用），折叠的结果是一个右值引用。，引用折叠规则如下：

![](https://res.cloudinary.com/bytedance14/image/upload/v1546336609/blog/20170903164922085.png)

与模板配合使用的例子:
```
#include<iostream>
using namespace std;
void runcode(int && m)
{
    cout << "rvalue ref" << endl;
}
void runcode(int & m)
{
    cout << "lvalue ref" << endl;
}
void runcode(const int & m)
{
    cout << "const lvalue ref" << endl;
}
void runcode(const int && m)
{
    cout << "const rvalue ref" << endl;
}

template <typename T>
void PerfectForward(T &&t)
{
    runcode(forward<T>(t));
}

int main()
{
    int a;
    int b;
    const  int c = 1;
    const int d = 0;
    PerfectForward(a);
    PerfectForward(move(b));
    PerfectForward(c);
    PerfectForward(move(d));
}
```
