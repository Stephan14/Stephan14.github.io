---
layout:     post
title:      "NULL与nullptr"
subtitle:   "C++ 空指针"
date:       2018-04-29 13:00:00
author:     "邹盛富"
header-img: "img/nullptr.jpg"
tags:
    - C/C++
---

### C中的NULL

一直很好奇C++为什么有NULL和nullptr两种值，所以就自己查了一些资料研究一下。原来早在1972年，C语言诞生的初期，常数0带有常数及空指针的双重身分。C使用preprocessor macro NULL表示空指针，让NULL及0分别代表空指针及常数0,在C语言中NULL实际被定义如下：
```
#define NULL ((void *)0)
```
也就是说`NULL`实际上是一种`void *`类型的指针，然后吧`void *`指针赋值给`int *`或者`foo_t *`的指针的时候，隐式转换成相应的类型。但是当使用C++编译器的时候，这样做是会出错的，因为C++是一种*强类型*的语言,这就导致`void *`类型不能够隐式转换成其他的指针类型，所以，编译器会提供如下的头文件定义`NULL`:

```
#ifdef __cplusplus ---简称：cpp c++ 文件
#define NULL 0
#else
#define NULL ((void *)0)
#endif
```

### C++中的NULL

其实在C++11中没有引入`nullptr`之前，由于C++中不能将`void *`类型的指针隐式转换成其他指针类型,而又为了解决空指针的问题，在C++中使用0来表示空指针(其实`NULL`就是0)，但是还是有如下的问题。比如我现在在一个类中定义了如下的函数：
```
void bar(sometype1 a, sometype2 *b);
```
同时在另外的函数中需要调用这个函数，调用方式如下：
a文件
```
bar(a, b);
```
b文件
```
bar(a, 0);
```
上述代码可以安全的运行，但是，如果我需要对这个类型进行扩展，重载其中的函数，重载后的函数声明如下：
```
void bar(sometype1 a, sometype2 *b);
void bar(sometype1 a, int i);
```
这个时候文件中的b文件中的函数并没有按照期望的方式运行着，它会调用`void bar(sometype1 a, int i)`函数，可以通过修改调用函数的参数类型来使其调用`void bar(sometype1 a, sometype2 *b)`函数，代码如下：
```
bar(a, static_cast<sometype2 *>(0));
```
但是上述的方式看起来是非常的别扭，并不是那么的优美，所以C++11中引入了`nullptr`来解决这个问题。


### C++11中的nullptr

关键词`nullptr`指代指针字面量，它是`std::nullptr_t`类型的纯右值。存在从`nullptr`到任何指针类型及任何指向成员指针类型的隐式转换，同样的转换对于任何空指针常量，空指针常量包含 `std::nullptr_t`的值，还有宏`NULL`。

如果引入了`nullptr`上述的问题便可以以很简单的方式来解决，只需要在调用的时候使用`nullptr`代替0就可以了，代码便可以按照期望的方式运行。如果某些编译器不支持C++11，可以自己模拟实现一个`nullptr`，代码如下：
```
const
class nullptr_t
{
public:
    template<class T>
    inline operator T*() const
        { return 0; }

    template<class C, class T>
    inline operator T C::*() const
        { return 0; }

private:
    void operator&() const;
} nullptr = {};
```
