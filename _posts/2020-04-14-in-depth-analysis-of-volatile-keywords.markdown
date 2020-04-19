---
layout:     post
title:      "C/C++ Volatile关键词深度剖析"
subtitle:   "Volatile关键词的三大特性"
date:       2020-04-14 09:42:30
author:     "zoushengfu"
header-img: "img/moss-4919093_1920.jpg"
tags:
    - C/C++
---

## 背景
最近写代码的时候使用到了volatile关键字，深入思考了一下volatile关键字的特性，并参照别人的文章验证了一下，所以在这里记录一下自己的验证结果。


## 特性

### 易变的
在介绍C/C++ Volatile关键词的”易变”性前，先让我们看看以下的两个代码片段，以及他们对应的汇编指令(以下用例的汇编代码，均为VS 2008编译出来的Release版本)。

测试用例一：非Volatile变量

![](https://res.cloudinary.com/bytedance14/image/upload/v1587309400/blog/v1.jpg)
b = a + 1;这条语句，对应的汇编指令是：lea ecx, [eax + 1]。由于变量a，在前一条语句a = fn(c)执行时，被缓存在了寄存器eax中，因此b = a + 1；语句，可以直接使用仍旧在寄存器eax中的a，来进行计算，对应的也就是汇编：[eax + 1]。


测试用例二：Volatile变量

![](https://res.cloudinary.com/bytedance14/image/upload/v1587309477/blog/v2.jpg)

与测试用例一唯一的不同之处，是变量a被设置为volatile属性，一个小小的变化，带来的是汇编代码上很大的变化。a = fn(c)执行后，寄存器ecx中的a，被写回内存：mov dword ptr [esp+0Ch], ecx。然后，在执行b = a + 1；语句时，变量a有重新被从内存中读取出来：mov eax, dword ptr [esp + 0Ch]，而不再直接使用寄存器ecx中的内容。

小结

从以上的两个用例，就可以看出C/C++ Volatile关键词的第一个特性：**易变性。所谓的易变性，在汇编层面反映出来，就是两条语句，下一条语句不会直接使用上一条语句对应的volatile变量的寄存器内容，而是重新从内存中读取。volatile的这个特性，相信也是大部分朋友所了解的特性。**


在了解了C/C++ Volatile关键词的”易变”特性之后，再让我们接着继续来剖析Volatile的下一个特性：”不可优化”特性。

### 不可优化
与前面介绍的”易变”性类似，关于C/C++ Volatile关键词的第二个特性：”不可优化”性，也通过两个对比的代码片段来说明：

测试用例三：非Volatile变量
![](https://res.cloudinary.com/bytedance14/image/upload/v1587309540/blog/v3.jpg)
在这个用例中，非volatile变量a，b，c全部被编译器优化掉了 (optimize out)，因为编译器通过分析，发觉a，b，c三个变量是无用的，可以进行常量替换。最后的汇编代码相当简介，高效率。

测试用例四：Volatile变量

![](https://res.cloudinary.com/bytedance14/image/upload/v1587309598/blog/v4.jpg)

测试用例四，与测试用例三类似，不同之处在于，a，b，c三个变量，都是volatile变量。这个区别，反映到汇编语言中，就是三个变量仍旧存在，需要将三个变量从内存读入到寄存器之中，然后再调用printf()函数。

小结


从测试用例三、四，可以总结出C/C++ Volatile关键词的第二个特性：**“不可优化”特性。volatile告诉编译器，不要对我这个变量进行各种激进的优化，甚至将变量直接消除，保证程序员写在代码中的指令，一定会被执行。** 相对于前面提到的第一个特性：”易变”性，”不可优化”特性可能知晓的人会相对少一些。但是，相对于下面提到的C/C++ Volatile的第三个特性，无论是”易变”性，还是”不可优化”性，都是Volatile关键词非常流行的概念。

### 顺序性
C/C++ Volatile关键词前面提到的两个特性，让Volatile经常被解读为一个为多线程而生的关键词：一个全局变量，会被多线程同时访问/修改，那么线程内部，就不能假设此变量的不变性，并且基于此假设，来做一些程序设计。当然，这样的假设，本身并没有什么问题，多线程编程，并发访问/修改的全局变量，通常都会建议加上Volatile关键词修饰，来防止C/C++编译器进行不必要的优化。但是，很多时候，C/C++ Volatile关键词，在多线程环境下，会被赋予更多的功能，从而导致问题的出现。

回到本文背景部分我的那篇微博，我的这位朋友，正好犯了一个这样的问题。其对C/C++ Volatile关键词的使用，可以抽象为下面的伪代码：

![](https://res.cloudinary.com/bytedance14/image/upload/v1587307000/blog/v5.jpg)
这段伪代码，声明另一个Volatile的flag变量。一个线程(Thread1)在完成一些操作后，会修改这个变量。而另外一个线程(Thread2)，则不断读取这个flag变量，由于flag变量被声明了volatile属性，因此编译器在编译时，并不会每次都从寄存器中读取此变量，同时也不会通过各种激进的优化，直接将if (flag == true)改写为if (false == true)。只要flag变量在Thread1中被修改，Thread2中就会读取到这个变化，进入if条件判断，然后进入if内部进行处理。在if条件的内部，**由于flag == true，那么假设Thread1中的something操作一定已经完成了，** 在基于这个假设的基础上，继续进行下面的other things操作。



通过将flag变量声明为volatile属性，很好的利用了本文前面提到的C/C++ Volatile的两个特性：”易变”性；”不可优化”性。按理说，这是一个对于volatile关键词的很好应用，而且看到这里的朋友，也可以去检查检查自己的代码，我相信肯定会有这样的使用存在。



但是，这个多线程下看似对于C/C++ Volatile关键词完美的应用，实际上却是有大问题的。问题的关键，就在于前面标红的文字：**由于flag = true，那么假设Thread1中的something操作一定已经完成了。** flag == true，为什么能够推断出Thread1中的something一定完成了？其实既然我把这作为一个错误的用例，答案是一目了然的：** 这个推断不能成立，你不能假设看到flag == true后，flag = true;这条语句前面的something一定已经执行完成了。** 这就引出了C/C++ Volatile关键词的第三个特性：顺序性。



同样，为了说明C/C++ Volatile关键词的”顺序性”特征，下面给出三个简单的用例 (注：与上面的测试用例不同，下面的三个用例，基于的是Linux系统，使用的是”GCC: (Debian 4.3.2-1.1) 4.3.2″)：

测试用例五：非Volatile变量
![](https://res.cloudinary.com/bytedance14/image/upload/v1587308983/blog/v6.jpg)
一个简单的示例，全局变量A，B均为非volatile变量。通过gcc O2优化进行编译，你可以惊奇的发现，A，B两个变量的赋值顺序被调换了！！！在对应的汇编代码中，B = 0语句先被执行，然后才是A = B + 1语句被执行。

在这里，我先简单的介绍一下C/C++编译器最基本优化原理：**保证一段程序的输出，在优化前后无变化。** 将此原理应用到上面，可以发现，虽然gcc优化了A，B变量的赋值顺序，但是foo()函数的执行结果，优化前后没有发生任何变化，仍旧是A = 1；B = 0。因此这么做是可行的。

测试用例六：一个Volatile变量
![](https://res.cloudinary.com/bytedance14/image/upload/v1587309109/blog/v7.jpg)
此测试，相对于测试用例五，最大的区别在于，变量B被声明为volatile变量。通过查看对应的汇编代码，B仍旧被提前到A之前赋值，Volatile变量B，并未阻止编译器优化的发生，编译后仍旧发生了乱序现象。

如此看来，**C/C++ Volatile变量，与非Volatile变量之间的操作，是可能被编译器交换顺序的。**

通过此用例，已经能够很好的说明，本章节前面，通过flag == true，来假设something一定完成是不成立的。在多线程下，如此使用volatile，会产生很严重的问题。但是，这不是终点，请继续看下面的测试用例七。

测试用例七：两个Volatile变量
![](https://res.cloudinary.com/bytedance14/image/upload/v1587309301/blog/v8.jpg)

同时将A，B两个变量都声明为volatile变量，再来看看对应的汇编。奇迹发生了，A，B赋值乱序的现象消失。此时的汇编代码，与用户代码顺序高度一直，先赋值变量A，然后赋值变量B。

如此看来，**C/C++ Volatile变量间的操作，是不会被编译器交换顺序的。**

### happens-before
通过测试用例六，可以总结出：C/C++ Volatile变量与非Volatile变量间的操作顺序，有可能被编译器交换。因此，上面多线程操作的伪代码，在实际运行的过程中，就有可能变成下面的顺序

![](https://res.cloudinary.com/bytedance14/image/upload/v1587309685/blog/v9.jpg)

由于Thread1中的代码执行顺序发生变化，flag = true被提前到something之前进行，那么整个Thread2的假设全部失效。由于something未执行，但是Thread2进入了if代码段，整个多线程代码逻辑出现问题，导致多线程完全错误。



细心的读者看到这里，可能要提问，根据测试用例七，C/C++ Volatile变量间，编译器是能够保证不交换顺序的，那么能不能将something中所有的变量全部设置为volatile呢？这样就阻止了编译器的乱序优化，从而也就保证了这个多线程程序的正确性。



针对此问题，很不幸，仍旧不行。将所有的变量都设置为volatile，首先能够阻止编译器的乱序优化，这一点是可以肯定的。但是，别忘了，编译器编译出来的代码，最终是要通过CPU来执行的。目前，市场上有各种不同体系架构的CPU产品，CPU本身为了提高代码运行的效率，也会对代码的执行顺序进行调整，这就是所谓的CPU Memory Model (CPU内存模型)。关于CPU的内存模型，可以参考这些资料：[Memory Ordering From Wiki](http://en.wikipedia.org/wiki/Memory_ordering)；[Memory Barriers Are Like Source Control Operations From Jeff Preshing](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)；[CPU Cache and Memory Ordering From 何登成](http://hedengcheng.com/?p=648)。下面，是截取自Wiki上的一幅图，列举了不同CPU架构，可能存在的指令乱序。

![](https://res.cloudinary.com/bytedance14/image/upload/v1587309823/blog/867145-20171117164556593-1798589645.png)

从图中可以看到，X86体系(X86，AMD64)，也就是我们目前使用最广的CPU，也会存在指令乱序执行的行为：StoreLoad乱序，读操作可以提前到写操作之前进行。



因此，回到上面的例子，哪怕将所有的变量全部都声明为volatile，哪怕杜绝了编译器的乱序优化，但是针对生成的汇编代码，CPU有可能仍旧会乱序执行指令，导致程序依赖的逻辑出错，volatile对此无能为力。



其实，针对这个多线程的应用，真正正确的做法，是构建一个happens-before语义。关于happens-before语义的定义，可参考文章：[The Happens-Before Relation](http://preshing.com/20130702/the-happens-before-relation/)。下面，用图的形式，来展示happens-before语义：

![](https://res.cloudinary.com/bytedance14/image/upload/v1587309914/blog/867145-20171117164617687-1593549798.png)

如图所示，所谓的happens-before语义，就是保证Thread1代码块中的所有代码，一定在Thread2代码块的第一条代码之前完成。当然，构建这样的语义有很多方法，我们常用的Mutex、Spinlock、RWLock，都能保证这个语义 (关于happens-before语义的构建，以及为什么锁能保证happens-before语义，以后专门写一篇文章进行讨论)。但是，C/C++ Volatile关键词不能保证这个语义，也就意味着C/C++ Volatile关键词，在多线程环境下，如果使用的不够细心，就会产生如同我这里提到的错误。



小结
C/C++ Volatile关键词的第三个特性：**”顺序性”，能够保证Volatile变量间的顺序性，编译器不会进行乱序优化。**Volatile变量与非Volatile变量的顺序，编译器不保证顺序，可能会进行乱序优化。同时，C/C++ Volatile关键词，并不能用于构建happens-before语义，因此在进行多线程程序设计时，要小心使用volatile，不要掉入volatile变量的使用陷阱之中。

## 参考

https://github.com/deepwaterooo/midiController/blob/master/docs/QThread.org
