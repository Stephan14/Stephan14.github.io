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

## 线程安全的对象生命周期管理

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

## 线程同步精要

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
