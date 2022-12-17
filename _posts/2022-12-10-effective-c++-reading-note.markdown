---
layout:     post
title:      "​effective c++ 改善程序与设计的55个具体做法"
subtitle:   "读书笔记"
date:       2022-12-10 21:42:30
author:     "邹盛富"
header-img: "img/WechatIMG11.jpeg"
tags:
    - C/C++
---


## 让自己习惯c++

### 条款03:尽可能使用const

以下2种写法意义相同
```
void f1(const Widget* pw);
void f2(Widget const * pw);
```

#### const和迭代器
```
std::vector<int> vec;

const std:::vector<int>::iterator iter = vec.begin();
*iter = 10;
++iter; //错误！！！！


std::vector<int>::const_iterator citer = vec.begin();
*citer = 10; //错误！！！！
++citer;
```

- mutable可以释放掉non-static成员变量的bitwise constness约束
- 当const和non-const的成员函数有着实质等价的实现时，令non-const版本调用const版本可避免代码重复

### 条款04:确定对象被使用前已先被初始化

- 为内置类型对象进行初始化，因为C++不保证初始化它们
- 构造函数最好使用成员初始化，而不要在构造函数本体内部使用赋值操作；初始化列表的成员变量，其排列次序应该和它们在class中的声明次序有关
- 为免除“跨编译单元之间的初始化次序”问题，使用local static 对象替换non-local static对象。

```
class FileSystem {};
FileSystem& tfs() {
    static FileSystem fs;
    return fs;
}

class Directory {};

Directory::Directory( params ) {
    std::size_t disks = tfs().numDisks();
}

Directory& tempDir() {
    static Directory td;
    return td;
}
```
但是上述使用方式在多线程的场景下会有问题，解决方案是在单线程启动的时候手工调用reference-returning函数。

## 构造/析构/赋值运算

### 条款05:了解C++默默编写并调用哪些函数
如果你没有声明的话，编译器会类声明一个copy构造函数、一个copy assignment操作符和一个析构函数（此时这个析构函数是non-virtual析构函数，除非这个class的base class自身声明有virtual析构函数）。此外，如果你没有声明任何构造函数，编译器也会为你声明一个default构造函数。

> 如果一个类中包含reference成员和const成员，必须自己定义copy assignment操作符。

### 条款06:若不想使用编译器自动生成的函数，就该明确拒绝
可以将copy构造函数或者copy assignment操作符声明为private，但是这样做不是特别安全，因为成员函数或者友元函数还是可以调用private函数，除非不去定义他，这样在程序链接的时候会报错。

### 条款07:为多态基类声明virtual析构函数
当derived class对象经由一个base class指针被删除，而该base class带有一个non-virtual析构函数，其结果是未定义的，实际执行时通常发生的是对象的derived成分没有被销毁。但是也不是意味着所有的class都需要将析构函数标记为virtual，因为每个携带virtual函数的class都有对应的vtbl，当对象调用某一个virtual函数时，实际上被调用的函数取决于该对象的vptr所指向的那个vtbl，编译器在其中寻找对应的函数，这就有可能改变一个对象的体积。所以只有当class内至少含有一个virtual函数，才为它声明virtual析构函数。


析构函数的运作方式是most derived的那个class其析构函数最先被调用，然后是其每个base class的析构函数被调用，构造函数与其相反。

### 条款08:别让异常逃离析构函数
C++并不禁止析构函数突出异常，但是它不鼓励你这样做。只要析构函数抛出异常，程序可能会过早结束或者出现不明确的行为。

### 条款09:绝不在构造函数和析构过程中调用virtual函数
base class构造期间virtual函数绝不会下降到derived class阶层，因为在执行base class的构造函数的时候，derived class的成员变量尚未初始化，如果可以下降到derived class阶层，必然会访问local变量，但是这些变量可能是未初始化的。

### 条款10:令operator=返回一个reference to *this

### 条款11:在operator=中处理“自我赋值”
- 确保当前对象自我赋值的时候operator=有良好的行为，包括比较“来源对象”和“目标对象”的地址，精心周到的语句顺序以及 copy and swap。
- 确保任何函数如果操作一个以上的对象，其中多个对象是同一个对象时，其行为仍然正确。

### 条款12:复制对象时勿忘其每个成分
- 当你编写一个copy构造函数或者copy assignment操作符时，你必须1）复制所有的成员变量2）调用所有base class内的所有的copy构造函数或者copy assignment操作符。
- 不要尝试以某个copy函数实现另一个copy函数，应该将共同机制放进第三个函数中，并由2个copy函数共同调用。

## 资源管理

### 条款13:以对象管理资源
主要思想：把资源放到对象内，依赖C++的“析构函数的自动调用机制”确保资源被释放。shared_ptr在其析构函数中做delete而不是delete[]动作，这意味着在动态分配的数组上使用shared_ptr是不正确的。

### 条款14:在资源管理类中小心copy行为

当一个RAII对象被复制时，一般会进行以下2种方式处理：
- 禁止复制
- 对底层资源使用“引用计数”，还有一些实现选择复制底部的资源或者转移底部资源的所有权

### 条款15:在资源管理类中提供对原始资源的访问

### 条款16:成对使用new和delete时要采取相同形式

- 尽量对于数组形式不要做typedef动作，避免在new完之后的delete操作出现未定义的行为
- 如果你在new表达式中使用[],必须在相应的delete表达式中也使用[];如果你在new表达式中没有使用[],一定不要相应的delete表达式中也使用[];

### 条款17:以独立语句将newed对象置入智能指针

