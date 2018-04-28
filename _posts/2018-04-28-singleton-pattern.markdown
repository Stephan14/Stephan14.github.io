---
layout:     post
title:      "单例模式"
subtitle:   "三种单例模式的C++实现"
date:       2018-04-18 09:09:00
author:     "邹盛富"
header-img: "img/singleton.png"
tags:
    - 设计模式
---

### 单例模式效果

通过单例模式，可以做到以下三点：
- 确保一个类只有一个实例被创建
- 提供了一个对对象全局访问的指针
- 在不影响单例类的客户端的情况下允许将来有多个实例

### 单例模式优缺点

#### 单例模式优点
- 跨平台：使用合适的中间件可以把Singleton扩展为跨多个计算机工作
- 适用于任何类：只要将初始化函数设为私有，并增加相应的静态函数和变量，就能把类变成Singleton
- 延迟性：如果Singleton从未使用，就不会创建（仅指懒汉模式）

#### 单例模式缺点
- 不变重用：在C++下需要定义模板才能实现Singleton的 重用。

- 通常单例模式的实现分为以下几种：

### 实现方式
- 懒汉模式
- 饿汉模式
- 多线程式（解决懒汉模式的问题）

### 懒汉模式

#### 使用场景
延迟加载，也就是说直到实力类被用到的时候才会被加载。使用懒汉模式时，Singleton在程序第一次调用的时候才会初始化自己，代码同上。使用该模式时，由于if语句的存在，会影响调用的效率。而且，在多线程环境下使用时，为了保证只能初始化一个实例，需要用锁来保证线程安全性，防止同时多个线程进入if语句中。如果遇到处理大量数据时，锁会成为整个性能的瓶颈。一般懒汉模式适用于程序一部分中需要使用Singleton，且在实例化后没有大量频繁访问或线程访问的情况。

#### 代码

Singleton.h文件
```
#ifndef __C__Review__Singleton__  
#define __C__Review__Singleton__  

#include <iostream>  
class Singleton{  

private:  

    Singleton() { }  
    Singleton(const Singleton&);//无实现  
    Singleton& operator=(const Singleton&); //无实现  

    static Singleton *instance;  

public:  

    static Singleton *getInstance();  
    static void release();  
};  
#endif /* defined(__C__Review__Singleton__) */  
```
Singleton.cpp文件
```
#include "Singleton.h"  
Singleton *Singleton::instance = 0;  

Singleton* Singleton::getInstance()  
{  
    if(instance == nullptr)  
        instance = new Singleton();  
    return instance;  
}  

void Singleton::release()  
{  
    if (instance != nullptr) {  
        delete instance;  
        instance = nullptr;  
    }  
}
```
main.cpp文件
```
#include "Singleton.h"  

using namespace std;  

int main(int argc, const char * argv[]) {  
    Singleton *s1 = Singleton::getInstance();  
    cout<<s1<<endl;  

    Singleton *s2 = Singleton::getInstance();  
    cout<<s2<<endl;  

    Singleton *s3 = Singleton::getInstance();  
    cout<<s3<<endl;  

    s1->release();  
    s2->release();  
    s3->release();  
    return 0;  
}
```

上述代码在退出的时候需要显示的调用`release()`函数，这样用起来比较麻烦，所以又存在第二种方式实现懒汉模式。
```
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton theSingleton;
        return theSingleton;
    }

private:
    Singleton();                 // ctor hidden
    Singleton(Singleton const&); // copy ctor hidden
    Singleton& operator=(Singleton const&); // assign op. hidden
    ~Singleton();                // dtor hidden
};
```
上述代码中在在`getInstance()`函数中定义局部静态变量，只会在第一次调用`getInstance()`函数时初始化，另外，`getInstance()` 返回的是对局部静态变量的引用, 如果返回的是指针，`getInstance()` 的调用者很可能会误认为他要检查指针的有效性，并负责销毁。构造函数和拷贝构造函数也私有化了，这样类的使用者不能自行实例化。

#### 注意
- 任意两个`Singleton`类的构造函数不能相互引用对方的实例, 否则会导致程序崩溃. 如:


```
    ASingleton& ASingleton::Instance() {
        const BSingleton& b = BSingleton::Instance();
        static ASingleton theSingleton;
        return theSingleton;
    }

    BSingleton& BSingleton::Instance() {
        const ASingleton & b = ASingleton::Instance();
        static BSingleton theSingleton;
        return theSingleton;
    }
```
- 多个`Singleton`实例相互引用的情况下，需要谨慎处理析构函数。 如：初始化顺序为`ASingleton->BSingleton->CSingleton`的三个`Singleton`类，其中`ASingleton BSingleton`的析构函数调用了`CSingleton`实例的成员函数，程序退出时，`CSingleton`的析构函数将首先被调用，导致实例无效，那么后续`ASingleton` `BSingleton`的析构都将失败，导致程序异常退出。

### 饿汉模式

#### 使用场景
使用饿汉模式时，Singleton在程序一开始就将自己实例化，之后的GetInstance方法仅返回实例的指针即可，这样就解决了上述提到的if语句影响效率的问题。饿汉模式适用于Singleton在程序运行过程中一直被频繁调用，这样由于预先加载了实例，访问实例时没有if语句，效率更高。但要注意到，如果Singleton的成员比较庞大、复杂，实例化Singleton会花一些时间，且这个实例一直占用着大量内存，在使用时要注意这部分的开销。使用饿汉模式用于多线程编程的话，由于线程访问之前，实例已存在，就不需要像懒汉模式中加入锁，因此饿汉模式保证了多线程安全。饿汉模式比较适用于程序整个运行过程中都需要访问、会被频繁访问或者需要被多线程访问的情况。

#### 代码
Singleton.h文件
```
#ifndef __C__Review__Singleton__  
#define __C__Review__Singleton__  

#include <iostream>  

class Singleton{  

private:  

    Singleton(){}  
    Singleton(const Singleton& other){}  

    Singleton& operator=(const Singleton& other);  

    static Singleton *instance;  

public:  

    static Singleton *getInstance();  

};  

#endif /* defined(__C__Review__Singleton__) */  
```
Singleton.cpp文件
```
#include "Singleton.h"  

Singleton *Singleton::instance = new Singleton();  

Singleton* Singleton::getInstance()  
{  
    return instance;  
}
```
main.cpp
```
#include "Singleton.h"  

using namespace std;  

int main(int argc, const char * argv[]) {  
    Singleton *s1 = Singleton::getInstance();  
    cout<<s1<<endl;  

    Singleton *s2 = Singleton::getInstance();  
    cout<<s2<<endl;  

    Singleton *s3 = Singleton::getInstance();  
    cout<<s3<<endl;  


    return 0;  
}
```

### 多线程式

#### 场景
为了解决懒汉模式在多线程情况下的线程安全，在懒汉模式的基础上进行了修改

#### 代码
```
class Lock  
{  
private:         
    CCriticalSection m_cs;  
public:  
    Lock(CCriticalSection  cs) : m_cs(cs)  
    {  
        m_cs.Lock();  
    }  
    ~Lock()  
    {  
        m_cs.Unlock();  
    }  
};  

class Singleton  
{  
private:  
    Singleton();  
    Singleton(const Singleton &);  
    Singleton& operator = (const Singleton &);  

public:  
    static Singleton *Instantialize();  
    static Singleton *pInstance;  
    static CCriticalSection cs;  
};  

Singleton* Singleton::pInstance = 0;  

Singleton* Singleton::Instantialize()  
{  
    if(pInstance == NULL)  
    {   //double check  
        Lock lock(cs);           //用lock实现线程安全，用资源管理类，实现异常安全  
        //使用资源管理类，在抛出异常的时候，资源管理类对象会被析构，析构总是发生的无论是因为异常抛出还是语句块结束。  
        if(pInstance == NULL)  
        {  
            pInstance = new Singleton();  
        }  
    }  
    return pInstance;  
}
```

参考链接

[如何正确地写出单例模式](http://wuchong.me/blog/2014/08/28/how-to-correctly-write-singleton-pattern/)

[C++ Singleton (单例) 模式最优实现](http://blog.yangyubo.com/2009/06/04/best-cpp-singleton-pattern/)
