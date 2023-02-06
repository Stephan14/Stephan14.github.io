---
layout:     post
title:      "​Effective C++ 改善程序与设计的55个具体做法"
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

显式转换函数
```
class Font {
public:
    FontHandle get() const { return f; }
};
```

隐式转换函数

```
class Font {
public: 
    operator FontHandle() const {
        return f;
    }
}
```

### 条款16:成对使用new和delete时要采取相同形式

- 尽量对于数组形式不要做typedef动作，避免在new完之后的delete操作出现未定义的行为
- 如果你在new表达式中使用[],必须在相应的delete表达式中也使用[];如果你在new表达式中没有使用[],一定不要相应的delete表达式中也使用[];

### 条款17:以独立语句将newed对象置入智能指针

## 设计与声明

### 条款18:让接口容易被正确使用，不易被误用
- “促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容
- “阻止误用”的办法包括建立新类型，限制类型上的操作，束缚对象值以及消除用户的资源管理责任

### 条款19:设计class犹如设计一个type

- 新type的对象应该如何创建和销毁？
这决定了构造函数和析构函数以及内存分配函数和释放函数的设计
- 对象的赋值和初始化有什么区别？
这决定了构造函数和赋值函数之间的得差异
- 新type的对象如果被以值传递，意味什么？
拷贝构造函数用来定义以值传递该如何实现
- 什么是新type的“合法值”？
成员函数（构造函数、赋值操作等）需要进行必要的错误检查
- 你的新type需要配合某个继承图系吗？
如果需要继承某些现有的class，则会受到这些class的影响，比如其virtual和non-virtual函数；如果允许其他的class继承你的type，则需要声明析构函数为virtual
- 新的type需要什么样的转换？
如果希望T1被隐式转换为T2类型，就必须在T1中写一个乐行转换函数（operator T2）或者在T2中写一个non-explicit-one-argument的构造函数。也可以参考条款15
- 什么样的操作符和函数对此type是合理的？
决定了你的函数声明哪些函数
- 什么样的标准函数应该被驳回？
这些函数必须声明为private
- 谁该取用新type的成员？
这决定了哪些成员为public，哪些为private，哪些为protected
- 什么是新type的“未声明接口”？
- 你的新type有多么的一般化？
是不是应该定义整个type家族？是不是应该定义一个class template？

### 条款20:宁以pass-by-reference-to-const替换pass-by-value
- 尽量以pass-by-reference-to-const替换pass-by-value。前者通常比较高效，可以避免继承时的切割问题
- 以上不适用内置类型、stl的迭代器和函数对象

### 条款21:必须返回对象时,别妄想返回其reference
绝不要返回pointer或者reference指向一个local stack对象，或者返回reference指向一个heap-allocated对象，或者返回一个pointer或者reference指向一个local static对象并且同时需要多个这样的对象

### 条款22:将成员变量声明为private
- 切记将成员变量声明为private，可以保证语法一致性（即所有访问该变量通过函数实现），可以划分访问控制（对该变量提供只读只写的功能）和提供弹性（验证约束条件，多线程同步控制等等）
- protected并不比public更具有封装性（比如删除一个变量的时候）

### 条款23:宁以non-member、non-friend替换member函数
越多的东西被封装，我们改变那些东西的能力越大，因为改变对应的代码只影响有限的用户。non-member、non-friend函数的封装性更大一些，因为它并不会增加“能够访问class内之private成分”的函数变量，但是这个论述只适用于non-member、non-friend函数，同时让函数变成non-member并不意味着它不可以是另一个class的member函数（例如一个工具类的static函数）同时，namespace和class不同, namespace可以跨越多个源码文件后者并不能. namespace是开放的，和class不同的是你可以在多个文件里面象同一个namespace里面添加东西，获得更少的编译依赖和增加扩展性。

### 条款24:若所有参数皆需类型转换，请为此采用non-member函数
如果需要为某个函数的所有参数（包括this指针所指向的那个隐喻参数）进行类型转换，那么这个函数必须是non-member，此时，编译器允许在每个实参身上执行隐式类型转换。

> 只有当参数被列于参数列内，这个参数才是隐式类型转换的合格参与者

### 条款25:考虑写出一个不抛出异常的swap函数

当缺省的swap实现的效率不足时，需要做以下事情

- 提供一个public swap成员函数，让它高效置换你的类型的两个对象值
- 在你的class或者template所在的命名空间内提供一个non-member swap,并令他调用上述成员函数
- 如果你正在编写一个class(而非 class template)为你的class特化std::swap,并令他调用你的swap成员函数
- 用户使用的时候，需要using::std以便让std::swap在你函数曝光内可见

> 特化必须在同一命名空间下进行，可以特化类模板也可以特化函数模板，但类模板可以偏特化和全特化，而函数模板只能全特化，但函数允许重载，声明另一个函数模板即可替代偏特化的需要

## 实现

### 条款26:尽可能延后变量定义式的出现时间

不只要延后变量的定义，直到非得使用该变量的前一刻为止，甚至应该尝试延后这份定义知道能给它初始值实参为止。

### 条款27:尽量少做转型动作

> 之所以要使用dynamic_cast，通常是因为想要在一个你任务是derived class对象上执行derived class操作函数，但是你只有一个指向base的pointer或者reference。

> static_cast 可以用户将non-const转化为const对象，int转换成double，void*指针转化成typed指针，将pointer-to-base转为pointer-to-derived等，但是不能将const转化为non-const(这个转换只有const_cast才能实现)

- 如果可以，尽量避免转型，在特别注重代码效率的场景下避免使用dynamic_cast
- 如果转型是必要的，试着将它隐藏在某个函数的背后，这样在用户调用该函数时，不需要将转型放到他们的代码中
- 建议使用c++-style转型，不要使用旧式转型

### 条款28:避免返回handles指向对象内部成分
避免返回handles（包括references、指针、迭代器）指向对象内部。遵守这个条件可增加封装性，帮助const成员函数的行为像一个const，并将发生“虚吊号码牌”的可能性将至最低

### 条款29:为“异常安全”而努力是值得的

- 异常安全函数即使发生异常也不会泄露资源或者允许任何数据结构败坏。这样的函数区分为三种可能保证：基本型、强烈型、不抛异常型
- "强烈保证"往往可以通过copy-and-swap实现出来，但是并非对所有的函数都可以实现或者具备实现的意义
- 函数提供的"异常安全保证"通常最高只等于其所有调用之各个函数的"异常安全保证"中的最弱者

### 条款30:透彻了解inlining的里里外外

inline是将“对此函数的每一个调用”都以函数本体替换，这样可能增加目标代码的大小，从而造成程序体积太大，导致额外的换页行为，降低指令高速缓存装置的集中率。

- 将大多数inlining限制在小型、被频繁调用的函数身上。这可使日后的调试过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化
- 不要只因为function template出现在头文件，就将他们声明为inline

### 条款31:将文件间的编译依存关系降低至最低

编译依赖的最小化的本质是让头文件尽可能自我满足，如果做不到，则让它与其他文件内的声明式相依而不是定义式相依。主要通过以下方式实现：
1. 如果使用object reference 或者object pointers可以实现，就不要使用objects（因为这种方式需要知道类型的定义式）
2. 如果能够，尽量以class声明式替换class定义式。当你声明一个函数并且用到某个class时，并不需要改class的定义，及时函数以by value方式传递该类型的参数或者返回值
3. 为声明式和定义式提供不同的头文件

基于上述思想，提供2种手段分别是*Handle class* 和*Interface class*
