---
layout:     post
title:      "​C++获取exception栈"
subtitle:   "使用GDB调试C++进程"
date:       2022-12-24 17:34:30
author:     "邹盛富"
header-img: "img/20221224-173648.jpg"
tags:
    - C/C++
---

## 问题
当C++程序core dump的时候，有可能是程序中抛出了exception，但是上层的代码没有进行catch 这种exception导致的。同时，如何定位这种抛出exception的地方比较困难，这里提供一种通用的方式来定位抛出exception的函数。

## 例子

```
#include <iostream>
#include <string>
// #include <exception>
#include <stdexcept>

void  ter_handler(){
     printf ( "custom handler\n" );
}

void  test(){
     throw  std::runtime_error( "test function" );
}

int  main( int  argc,  char ** argv)
{
     //std::set_terminate(__gnu_cxx::__verbose_terminate_handler);
     //std::set_terminate(ter_handler);

     //try {
     // throw 5;
     //throw  std::runtime_error( "test error" );
     //}
     //catch (...){
     //    printf ( "catch exception\n" );
     //}

     test();
     return  0;
}
```

通过如下命令编译改程序：
```
g++ -o exception exception.cpp -std=c++11
```

## 调试
主要的思路是通过
```
b main
catch throw
r
```
> 在程序执行之前，catch throw/catch是无效的，需要在程序执行之后(先在main处设置断点)，使用catch throw才有效

这3个命令来调试，过程如下：

```
➜  code gdb ./exception
GNU gdb (Debian 7.12-6) 7.12.0.20161007-git
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
/home/zoushengfu/.gdbinit:1: Error in sourced command file:
Undefined command: "add-shared-symbol-file".  Try "help".
Reading symbols from ./exception...(no debugging symbols found)...done.
(gdb) b main
Breakpoint 1 at 0xb69
(gdb) catch throw
Catchpoint 2 (throw)
(gdb) r
Starting program: /data00/home/zoushengfu/code/exception

Breakpoint 1, 0x0000555555554b69 in main ()
(gdb) bt
#0  0x0000555555554b69 in main ()
(gdb) c
Continuing.

Catchpoint 2 (exception thrown), 0x00007ffff7ae626d in __cxa_throw () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
(gdb) bt
#0  0x00007ffff7ae626d in __cxa_throw () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#1  0x0000555555554b4f in test() ()
#2  0x0000555555554b79 in main ()
(gdb) c
Continuing.
terminate called after throwing an instance of 'std::runtime_error'
  what():  test function

Program received signal SIGABRT, Aborted.
__GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
51      ../sysdeps/unix/sysv/linux/raise.c: No such file or directory.
(gdb) bt
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:51
#1  0x00007ffff71d142a in __GI_abort () at abort.c:89
#2  0x00007ffff7ae80ad in __gnu_cxx::__verbose_terminate_handler() () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#3  0x00007ffff7ae6066 in ?? () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#4  0x00007ffff7ae60b1 in std::terminate() () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#5  0x00007ffff7ae62c9 in __cxa_throw () from /usr/lib/x86_64-linux-gnu/libstdc++.so.6
#6  0x0000555555554b4f in test() ()
#7  0x0000555555554b79 in main ()
```
