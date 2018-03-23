---
layout:     post
title:      "c指针和数组"
subtitle:   "基础知识"
date:       2018-03-23 21:34:00
author:     "邹盛富"
header-img: "img/point.jpg"
tags:
    - C/C++
---

尝试写了一个小程序，但是运行时发生错误，就简单的回顾了一下C语言中的数组名和指针

- 相同点：

     他们都具有指针值，都可以通过下标引用和间接访问操作。

- 不同点：

    声明一个数组的时候，编译器先根据指定的元素的数量分配内存空间容纳数组元素，在创建数组名，注意数组名是一个指针常量。

这两种声明只有当他们是函数的形参的时候才是相等的。

声明一个指针的时候，编译器为指针本身分配一个内存空间用于存储指针值。

下面是这个小程序修改之后的正确的版本以及部分注释

```
#include<stdio.h>  
int main(int argc, char const *argv[]) {  
  char str[] = "sdfdsgfd";//此处不能够使用指针  
  char *pstr = str;  

  while (*pstr) {//也可以使用 *pstr != '\0'  
    printf("%s\n", pstr++ );  
    //str++;数组名为常量指针，不能使用str++  
  }  

  return 0;  

```

#### 参考连接

[C数组与指针](https://blog.csdn.net/loveyou426/article/details/7901177)

[ extern int \*a与int a\[\] ](https://blog.csdn.net/wdkirchhoff/article/details/40989321)
