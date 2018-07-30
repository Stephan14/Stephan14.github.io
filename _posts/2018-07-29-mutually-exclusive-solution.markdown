---
layout:     post
title:      "互斥解决方案"
subtitle:   "硬件和软件方法"
date:       2018-07-29 15:36:00
author:     "邹盛富"
header-img: "img/waves-1867285_1920.jpg"
tags:
    - 操作系统
---

### 介绍

互斥是指进程之间的竞争关系

### 临界区的使用原则
1. 没有进程在临界区时，想进入临界区的进程可以进入
2. 不允许两个进程同时处于临街区
3. 临界区外运行的进程不得阻塞其他进程进入临界区
4. 不得使进程无限期等待进入临界区


### 解决方案
#### 软件解决方案
- Dekker解法
- Peterson解法

#### 硬件方案
- 屏蔽中断
- TSL指令

### 几种软件解决方法

#### 第一种

![](http://res.cloudinary.com/bytedance14/image/upload/v1532850068/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-29_%E4%B8%8B%E5%8D%883.37.23.png)

当P进程在`while(free)`之后并且`free=true`语句之前完成时间片执行，被调度下CPU，此时Q进程此时被调度到CPU上执行并且进入临界区，但是没有完全执行完临界区就完成时间片被调度下CPU后，P进程又被调度上CPU继续执行进入临界区，此时有两个进程处在临界区中，违反了临界区的使用原则

#### 第二种
![](http://res.cloudinary.com/bytedance14/image/upload/v1532850511/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-29_%E4%B8%8B%E5%8D%883.47.41.png)

如果Q进程不想进入临界区，则会导致P一直等待，不能进入临界区，违反了临界区的使用原则

#### 第三种
![](http://res.cloudinary.com/bytedance14/image/upload/v1532850812/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-29_%E4%B8%8B%E5%8D%883.52.59.png)

当P进程执行完`pturn=true`之后被调度下CPU，然后Q进程被调度上CPU执行`qturn=true`之后也被调度下CPU，此时P进程将会一直执行`while(qturn)`导致不能进入到临界区

### DEKKER算法
![](http://res.cloudinary.com/bytedance14/image/upload/v1532853576/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-29_%E4%B8%8B%E5%8D%884.39.03.png)
引入turn变量来标识应该哪个进程执行。

#### 缺点
1. 就算不使用临界区也会阻塞别的进程
2. 一直在测临界区是否是否空闲，浪费处理器资源
3. 只能处理固定顺序的进程序列

### PETERSON算法
![](http://res.cloudinary.com/bytedance14/image/upload/v1532854137/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-29_%E4%B8%8B%E5%8D%884.47.31.png)

#### 缺点

- 临界区范围大，不能保证所有指令并发执行
- 开关中断只能作用于一个处理器，所以适用于单处理器
- 用户程序不能使用特权指令



### 屏蔽中断
#### 开关中断指令
![](http://res.cloudinary.com/bytedance14/image/upload/v1532854368/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-29_%E4%B8%8B%E5%8D%884.52.18.png)

### TSL指令

TSL指令其工作如下所述：它将一个存储器字读到一个寄存器中，然后在该内存地址上存一个非零值。读数和写数操作保证是不可分割的—即该指令结束之前其他处理机均不允许访问该存储器字。执行TSL指令的CPU将锁住内存总线以禁止其他CPU在本指令结束之前访问内存。

为了使用TSL指令，我们必须用一个共享变量lock来协调对共享内存的访问。当lock为0时，任何进程都可以使用TSL指令将其置为1并读写共享内存。当操作结束时，进程用一条普通的MOVE指令将lock重新置为0。

![](http://res.cloudinary.com/bytedance14/image/upload/v1532927448/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-30_%E4%B8%8B%E5%8D%881.04.07.png)

如上图所示，其中展示了使用四条指令的汇编语言例程。第一条指令将lock原来的值拷贝到寄存器中并将lock置为1，随后这个原先的值与0相比较。如果它非零，则说明先前已被上锁，则程序将回到开头并再次测试。经过或长或短的一段时间后它将变成0(当前处于临界区中的进程退出临界区时)，于是子例程返回，并上锁。清除这个锁很简单，程序只需将0存入lock即可，不需要特殊的指令。

进程在进入临界区之前先调用enter_region。这将导致**忙等待**，直到锁空闲为止。随后它获得锁变量并返回。在进程从临界区返回时它调用leave_region，这将把lock置为0。

### XCHG指令

![](http://res.cloudinary.com/bytedance14/image/upload/v1532927772/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-07-30_%E4%B8%8B%E5%8D%881.15.47.png)


### 参考资料

[Dekker算法与Peterson算法-处理两个进程间互斥](https://dcrozz.github.io/2017/04/28/Dekker%E7%AE%97%E6%B3%95%E4%B8%8EPeterson%E7%AE%97%E6%B3%95-%E5%A4%84%E7%90%86%E4%B8%A4%E4%B8%AA%E8%BF%9B%E7%A8%8B%E9%97%B4%E4%BA%92%E6%96%A5/)
