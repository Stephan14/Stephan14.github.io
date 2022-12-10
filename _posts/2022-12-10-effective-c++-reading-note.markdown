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

### 条款03:尽可能石永红const

以下2种写法意义相同
```
void f1(const Widget* pw);
void f2(Widget const * pw);
```

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

