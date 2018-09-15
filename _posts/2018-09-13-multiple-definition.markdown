---
layout:     post
title:      "C++ 编译连接遇到的问题"
subtitle:   "C++ multiple definition"
date:       2018-09-13 21:05:30
author:     "邹盛富"
header-img: "img/starry-night-1149815_1920.jpg"
tags:
    - C++
---
## 背景
最近尝试编译C++项目，编译的时候遇到如下报错：
```
.obj/Inotify.o:(.bss+0x0): multiple definition of `EVENT_NAME'
.obj/FileServer.o:(.bss+0x0): first defined here
.obj/server_main.o:(.bss+0x0): multiple definition of `EVENT_NAME'
.obj/FileServer.o:(.bss+0x0): first defined here
collect2: error: ld returned 1 exit status
Makefile:71: recipe for target 'server_main' failed
make: *** [server_main] Error 1
```
回头看代码发现头文件`Inotify.h`这个文件在`FileServer.h`中已经被引用，然后再`server_main.cc`文件中又引用了`FileServer.h`这个文件,但是在最后执行`g++ .obj/Server.grpc.pb.o .obj/Server.pb.o .obj/Prepare.o .obj/FileServer.o .obj/Thread.o .obj/Inotify.o .obj/server_main.o -L /usr/local/lib   -std=c++11 `pkg-config --libs protobuf grpc++ grpc` -Wl,--no-as-needed -lgrpc++_reflection -Wl,--as-needed -ldl  util/.obj/File.o  -o server_main`命令进行链接的时候，将所有的目标文件装载到同一环境的时候出现了重复定义的问题，后来尝试将变量`EVENT_NAME`变成static就好了，于是就研究了一下出现这种问题的原因。

## 编译、链接过程
- 预处理将伪指令（宏定义、条件编译、和引用头文件）和特殊符号进行处理
    - 预处理程序将include头文件的内容包含进源文件，这个过程完成后，头文件就没用了
- 编译过程通过词法分析、语法分析等步骤生成汇编代码的过程，过程中还会进行优化
- 汇编过程将汇编代码翻译为目标机器指令的过程（.o文件，至少包含代码段和数据段）
- 链接程序将所有需要用到的目标代码（变量函数或其他库文件等）装配到一个整体中（可分为静态链接和动态链接）


## 问题
如果在头文件中定义了变量（是定义不是声明），并分别在a.c和b.c中进行了引用，编译过程中这个变量的符号会同时包含在a.o和b.o中，导致链接失败，原因是C语言规定“一个变量可以多次声明但只能定义一次”，解决办法是在头文件中加上#ifndef X条件编译，使该变量只定义一次，但是这里又有一个问题，该解决办法只适用C而不适用C++，在C++中，即使在头文件中加了#ifndef X，链接错误同样会发生，原因是C++中#ifndef X的作用域**仅在单个文件中**，因此只要在.h中定义了变量并在不同.cpp中进行引用，链接时都会报重定义错误，再说得直白点，a.cpp和b.cpp都引用了条件编译的g.h，g.h的条件编译只能分别保证在a.cpp和b.cpp中不出现重复定义，但在链接a.o和b.o的过程中就会发现重复定义。

下面是一个错误的例子：
```
#ifndef __CONST_H__
#define __CONST_H__

const char *zutypes[] = {

    "CL", "CY", "GM", "SSD", "XC", "ZS", "ZWX", "LS"
    , "KQWR", "LY", "KT", "DY", "FS", "GJ", "HC"
    , "JT", "LK", "YS", "MF", "YSH", "PJ", "FFZ"
    , "HZ", "TGWD", "FH", "XQ", "YD", "YH"

};   // 28种指数类型映射表

#endif // __CONST_H__

// hfTrans.h
#ifndef __HFTRANS_H__
#define __HFTRANS_H__

#include "const.h"

#endif // __HFTRANS_H__

// hfTrans.cpp
#include "hfTrans.h"
...

// main.cpp
#include "hfTrans.h"
...
```

编译输出:
```
Linking console executable: bin/Debug/main
obj/Debug/main.o:(.data+0x0): multiple definition of `zutypes'
obj/Debug/hfTrans.o:(.data+0x0): first defined here
collect2: ld 返回 1
Process terminated with status 1 (0 minutes, 0 seconds)
0 errors, 0 warnings
```

## 解决方法
### 变量前用static修饰
static限制了变量的作用域，该变量仅在引用.h的源文件中有效，也就是说.h被引用了几次这个变量就被定义了几次，且各变量之间互不影响（各变量具有不同的内存地址）。这种方法不适用于定义全局变量，因为它们不是同一个变量（相当于多个同名的人住在不同的地方）。

例子：
```
// global.h
#ifndef __GLOBAL_H__
#define __GLOBAL_H__

#include <stdio.h>

static int var = 10;

#endif

// test1.cpp
#include "global.h"

void print1()
{
    printf("%p = %d\n", &var, var);    // 打印变量的内存地址
}

// test2.cpp
#include "global.h"

void print2()
{
    printf("%p = %d\n", &var, var);
}

// main.cpp
#include "global.h"

extern void print1();
extern void print2();

int main()
{
    print1();

    var = 5;
    printf("%p = %d\n", &var, var);

    print2();

    return 0;
}
```

输出结果：
```
0x8049840 = 10
0x804983c = 5
0x8049844 = 10  # var地址各不相同，内容互不影响
Process returned 0 (0x0)   execution time : 0.046 s
Press ENTER to continue.
```

根据static的上述特性，在源文件开头处（紧跟include后）可直接定义static非全局变量。

### 变量前用const修饰
表示此变量是常量，内容不可修改，与static特性相似，该常量仅在引用.h的源文件中有效。将上述例子中的static关键字修改为const，可以发现每个源文件的var地址依然不同，因此这种方法也不适用于定义全局变量（当然，在某种程度上，如果不在乎重复分配内存也可以用这种方法）。

例子：
```
// global.h
#ifndef __GLOBAL_H__
#define __GLOBAL_H__

#include <stdio.h>

const int var = 10;

#endif
```
输出结果：
```
0x80485e0 = 10
0x80485d0 = 10
0x80485f0 = 10
Process returned 0 (0x0)   execution time : 0.005 s
Press ENTER to continue.
```
到这里可以发现在C++中，**const和static一样都可以使变量具有内部链接属性**。只有变量的作用域为当前模块时，该变量才可以在头文件中定义，否则链接时就会报重定义错误，因此只有const和static变量可以在头文件中定义。另外在C++中，const值在编译期间被保存在符号表中，即使在运行期间通过间接方法改变了const值（改变的其实是内存中的拷贝），输出值也不会改变。

根据const的上述特性，在源文件开头处（紧跟include后）可直接定义const非全局常量。

定义一般常量没有问题，需要注意的是**用const定义指针，指针必须符合上述原则才能通过链接**：
```
// global.h

const char str[][8] = { "Hello, ", "World!" };          // 正确, str是常量字符串数组
char const str[][8] = { "Hello, ", "World!" };          // 正确, 同上
static char str[][8] = { "Hello, ", "World!" };         // 正确
static const char str[][8] = { "Hello, ", "World!" };   // 正确

const char* str[] = { "Hello, ", "World!" };            // 错误，str非内部链接
char* const str[] = { "Hello, ", "World!" };            // 正确，但不建议常量字符串到char*的转换
const char* const str[] = { "Hello, ", "World!" };      // 正确, str是指向常量字符串的常量指针数组
static char* str[] = { "Hello, ", "World!" };           // 正确，但不建议

```

### 定义全局常量时经常将const和extern结合使用
前面提到const修饰的变量具有内部链接属性，用extern修饰的变量具有外部链接属性，也就是说将两者结合就可以实现全局和只读变量的目的，但需要说明的是，变量必须在头文件中给出声明而不是定义，然后在与头文件对应的源文件中给出定义（也可以在任意引用该头文件的源文件中给出定义，但不推荐）。

```
// global.h
#ifndef GLOBAL_H_INCLUDED
#define GLOBAL_H_INCLUDED

#include <stdio.h>

extern const int var;       // 声明var

#endif // GLOBAL_H_INCLUDED

// global.cpp
#include "global.h"

const int var = 10;     // 正确，定义var

// test1.cpp
#include "global.h"

//const int var = 10;       // 正确，但不推荐，容易出现重定义

void print1()
{
    const int var = 0;             // 错误，var的作用域为print1()
    printf("%p = %d\n", &var, var);    // 局部变量var覆盖了全局变量
}

// test2.cpp
#include "global.h"

void print2()
{
    printf("%p = %d\n", &var, var);
}

// main.cpp
#include "global.h"

extern void print1();
extern void print2();

int main()
{
    print1();

    printf("%p = %d\n", &var, var);

    print2();

    return 0;
}

```
输出结果：
```
0xbfcff14c = 0
0x80485d0 = 10
0x80485d0 = 10  # var地址相同
Process returned 0 (0x0)   execution time : 0.014 s
Press ENTER to continue.
```

可以看到在global.cpp中定义的var具有全局唯一性，**在每个模块中访问的var地址都相同**，例子的var是常量，不能改变它的值，如果在头文件中声明extern int var并在源文件中定义int var = 10，然后在需要用到var的模块中引入该头文件，就可以实现C语言的全局变量，并且它的值可以被改变。
