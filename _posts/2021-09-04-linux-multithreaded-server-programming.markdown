---
layout:     post
title:      "Linux服务端多线程编程"
subtitle:   "读书笔记"
date:       2021-09-04 08:42:30
author:     "zoushengfu"
header-img: "img/st-petersburg-4805295.jpeg"
tags:
    - C/C++
---

## 第一章:线程安全的对象生命周期管理

### 构造函数与多线程
构造函数做到多线程安全，唯一要求是在构造期间不要泄露this指针，即：
- 在构造期间不要注册任何回调函数
- 不要在构造期间把this指针传给跨线程对象
- 及时在构造函数最后一行也不可以，因为此类可能是基类

通常使用方式是二段式构造，即构造函数+initilalize()函数

### 析构函数遇到多线程存在问题
1. 其他线程函数是不是在访问即将析构对象？
2. 如果保证在执行成员函数时期对象会不会被另一个线程析构？
3. 在执行成员函数时，其析构函数会不会执行到一半

> 作为数据成员的mutex不能保护析构函数，因为作为成员变量的mutex其生命周期与对象的生命周期相同，析构操作可能发生在对象身亡之时和身亡之后；其次，对于基类，当调用起析构函数时，子类已经析构了，则mutex不能保证整个析构过程

解决上述的方案是使用shared_ptr

### 神奇shared_ptr
出现内存错误的几个方面：
- 缓冲区溢出：使用std::vector<char>/std::string或者自定义的Buffer class来管理缓冲区
- 野指针：使用shared_ptr
- 重复释放：使用scoped_ptr
- 内存泄漏：使用scoped_ptr
- 不配对的new[]/delete: 把new[]替换为std::vector或者scoped_array

### 再论shared_ptr的线程安全
shared_ptr不是100%线程安全的，但是他的引用计数是安全无锁的，但是对象的读写则不是。shared_ptr的线程安全和内建类型是一样的，即：
- 一个shared_ptr对象实体可以被多个线程同时读取
- 两个shared_ptr对象可以被2个线程同时写入（析构算是写入）
- 如果多个线程同时读取一个shared_ptr对象，则需要加锁（在多核系统上可以考虑使用spin lock）
> 以上是shared_ptr对象本身线程安全，不是它管理的对象的线程安全级别

shared_ptr作为函数参数传递时不洗复制，用reference to const作为参数类型即可

### shared_ptr技术与陷阱
#### 意外延长对象的声明周期
shared_ptr允许拷贝构造函数和赋值函数，如果一不小心遗留了一个拷贝，那么对象的生命周期可能就会延长。比如使用std::bind时如果实参为shared_ptr，则实参会被拷贝一份。

#### 函数参数
作为函数参数时，由于需要引用计数，因此拷贝的成本比原始指针要高，因此一般使用referenc to const作为参数类型

#### 析构函数在创建的时候被捕获

- shard_ptr<void>可以持有任何的对象，而且能安全的释放
- shared_ptr对象可以跨模块边界
- 二进制兼容，即使独显的大小变了，旧的client仍然可以使用新的动态库
- 析构函数可以定制

#### 析构所在的线程
最后一个指向对象的shared_ptr离开其作用域时，该对象会在同一个线程析构掉，未必是对象创建的线程

#### 现成的RAII handle
shared_ptr作为现成的handle对象，需要避免**循环引用**，通常做法是owner持有指向child的shared_ptr, child持有指向owner的weak_ptr

### 对象池
当时使用std::bind并且传入this指针时，需要注意该对象是否有可能析构。为了解决这个问题引入enable_shared_from_this, 该类继承enable_shared_from_this并且使用shared_from_this()代替this指针

## 第二章:线程同步精要

### 互斥锁
#### 只使用非递归的mutex
#### 死锁
- 单线程死锁：抽象出无锁调用的公共方法
- 多线程死锁

### 条件变量

对于wait端：
1. 必须与mutex一起使用，布尔表达式的读写需要mutex保护
2. 在mutex上锁的时候才能调用wait
3. 把判断布尔条件和wait()放到while循环中

对于signal/broadcast端：
1. 不一定要在mutex已上锁的情况下调用signal
2. 在single之前需要修改布尔表达式
3. 修改布尔表达式通常需要mutex保护
4. broadcast表示状态变化，signal表示自愿可用

### 不要使用读写锁和信号量
见到一个数据读多写少，使用读写锁未必正确：
1. 在持有read lock的情况下，有可能在维护阶段修改了其中的状态
2. 从性能角度来看，在read lock加锁阶段的开销不会比mutex lock小，因为其需要维护reade数据
3. read lock有可能提升为write lock，也有可能不提升
4. read lock可重入，但是write lock不可重入，为了防止write lock饥饿，write lock会阻塞read lock,导致read lock在重入的时候死锁（旧的read lock没有释放锁，又有write lock存在，导致新的read lock不能加锁）

信号量存在的问题:
- semaphore has no notion of ownership
- 其有自己的计数值，与代码的数据结构冲突，可能出错

### 线程安全的Singleton实现
```
template<typename T>
class Singleton : boost::noncopyable
{

public:
    static T& instance()
    {

        pthread_once(&ponce_, &Singleton::init);//只被初始化一次
        return *value_;
    }

private:
    Singleton();
    ~Singleton();
    static void init()
    {

        value_ = new T();
    }

    static pthread_once_t ponce_;
    static T* value_;
};

template<typename T>
pthread_once_t Singleton::ponce_ = PTHREAD_ONCE_INIT;

template<typename T>
T* Singleton<T>::value_ = NULL;
```

>c++11内存模型

### sleep不是同步原语

生产代码中等待可分为2种：
1. 等待资源可用，等待select、epoll_wait或者等待条件变量上
2. 等待进入临界区（等待mutex上）,通常时间比较短

### 借shared_ptr实现copy-on-write

- 对于write端，进入临界区内之后，如果shared_ptr的引用计数为1，则直接修改；如果引用不为1，则reset shared_ptr并new新的内存空间复制给shared_ptr，最后退出临界区
- 对于read端，进入临界区内之后，拷贝shared_ptr，退出临界区。在临界区外进行读操作

## 第十章:C++编译链接模型精要

c++至今没有模块机制，不能使用import或者using引入依赖的库，必须使用include头文件来机械地将库的接口声明以文本替换的方式载入，重新parse一遍。存在以下2个问题：

- 头文件具有传递性，引入不必要的依赖
- 头文件是在编译时使用，库文件在运行时使用，二者的时间差可能导致二进制不兼容

### C语言的编译模型及其成因

#### 为什么C语言需要预处理
为了减少内存使用的情况实现分离编译，C语言采用“隐式函数声明”的做法，代码在使用前文未定义的函数时，编译器不需要检查函数原型，会出现连接错误而非编译错误。为了减少各个源文件的重复代码，同时提高代码的可移植性，将公共信息坐成头文件，程序include用到的头文件即可。

#### C语言的编译模型

由于不能将整个语法树保存在内存中，C语言按照单遍编译的方式来设计。编译器只能看到目前已经解析过的代码，但不到之后的代码。所以：
- 结构体必须先定义，才能访问其成员，否则不知道成员的类型和偏移量，不能生成目标代码
- 局部变量有需要先定义再使用，否则不知道类型已经在stack中的位置，不能生成目标代码
- 对于外部变量，编译器只需要知道类型和名字，不要其地址。在生成目标代码的地址是个空白，留给链接器去填
- 编译器看到一个函数调用时按照隐式函数声明原则，可以生成调用函数的汇编代码，但是函数的实际地址不确定，留给链接器去填


### C++的编译模型

#### 单遍编译
C++也继承了单遍编译，编译器只能根据目前看到的代码作出决策，即使读到后面的代码也不会影响前面作出的决定。

```
void foo(int) {
    printf("foo(int);\n");
}

void bar() {
    foo('a'); //调用 foo(int)
}

void foo(char) {
    printf("foo(char)"); //修改foo(char)和foo(int)位置结果不一样
}
int main() {
    bar();
}
```

#### 向前声明

为了完成语法检查生成目标代码，编译器需要知道函数的参数个数和类型以及返回值类型，并不需要知道函数体的实现（除非是inline函数）。需要在某个源文件中定义这个函数，斗则会链接错误。如果在定义的时候把函数参数类型写错了，则在生成目标的文件的时候不会报错，但是在链接的时候会报错。

以下2中情况可以向前声明：
- 定义或者声明类的指针或者引用，包括用于函数参数、返回类型、局部变量、类型成员变量
- 声明一个类为参数或者返回类型的函数，但是如果要调用这个函数就需要知道类的定义，因为需要使用拷贝构造函数和析构函数

### C++链接
c++相比于c语言的连接，主要增加了两项：
- 函数重载,需要类型安全的连接(name mangling)
- vague linkage,同一个符号有多份互不冲突的定义

#### 函数重载
返回类型不参加函数重载

#### inline函数

如果编译器无法展开inline函数，则每个编译单元都会生成一个inline函数的目标代码。然后链接器会从多份实现中任选一个保留，其余的会丢弃。同时编译器自动生成的class的析构函数也是inline函数。

#### 模板

- 模板编译连接的不同之处在于，以上具有external linkage的对象通常会在多个单元被定义，链接器必须进行重复代码的消除，才可以生成正确的可执行文件。但是在-O2编译下，编译器会把短函数都inline展开。
- 对于private成员函数模板，只需要在头文件中给出声明，因为用户代码不会调用它，无法实现随意具现化，不会造成连接错误

#### 虚函数

一个多态class的vtable应该恰好被一个目标文件定义，但是C++无法判断是不是应该在当前编译单元中生成vtable定义，所以为每个编译单元都生成vtable，交给连接器去消除重复数据。如果我们不希望vtable导致目标文件膨胀，在头文件的class定义中声明out-line 虚函数


## 第十一章：反思C++面向对象与虚函数

### 朴实的C++设计

### 程序库的二进制兼容

#### 什么是二进制兼容性
在升级库文件的时候，不需要重新编译使用了这个库的可执行文件或者其他库文件，并且程序的功能不被破坏。

#### 有哪些情况会破坏库的ABI

C++编译器ABI主要包含以下几个方面：
- 函数参数传递方式，比如：x86-64用寄存器来传函数的前4个参数
- 虚函数的调用方式，通常是vptr/vtbl机制，然后用vtabl[offset]来调用
- struct 和class的内存布局，通过偏移量来访问数据成员
- name mangling
- RTTI和异常处理的实现

编译器如何判断一个改动是不是二进制兼容的，主要看旧的头文件是不是和新版本的动态库的使用方法兼容，因为新的库必然新的头文件，现在的二进制可执行文件还是按照旧的头文件中的方式来调用。

二进制不兼容的例子：
- 定义的non-virtual函数`void foo(int)`，参数类型改成了`double`，则会出现`undefined symbol`错误，因为mangled name不同。但是对于virtual函数，修改其参数类型，则不会出现错误，因为虚函数的决议通过便宜量，并不是靠符号名。
- 给函数添加默认参数，现有的可执行文件无法传这个额外的参数
- 增加虚函数，会造成vtbl里面的排列变化，不考虑只在莫问增加这种技巧
- 增加默认模板类型参数，这回导致name mangling发生变化
- 增加enum的值，这会早上错误，在末尾添加除外
- 给class增加数据成员，通常是不安全的。但是如果代码里面不是通过new 构造对象，而是通过factory返回对象指针或者shared_ptr，可能是安全；或者不需要用到data member的offset，也可能是安全的

#### 哪些做法多半是安全的
- 增加新的class
- 增加non-virtual成员函数或者static成员函数
- 修改数据成员的名称，因为二进制代码是按照偏移量来访问的


#### 反面教材：COM


#### 解决方法

- 采用静态链接，完全从源码编译出可执行文件
- 通过动态库的版本来管理控制兼容性
- 用pimpl技法，在头文件中只暴露non-virtual接口，并且class的大小是固定的