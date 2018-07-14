---
layout:     post
title:      "C++反射机制"
subtitle:   "可变参数模板实现C++反射"
date:       2018-07-14 11:53:00
author:     "铁芒箕"
header-img: "img/dawn-190055_1280.jpg"
tags:
    - C++
---
### 概要

本文描述一个通过C++可变参数模板实现C++反射机制的方法。该方法非常实用，在Nebula高性能网络框架中大量应用，实现了非常强大的动态加载动态创建功能。Nebula框架在Github的仓库地址。

C++11的新特性--可变模版参数（variadic templates）是C++11新增的最强大的特性之一，它对参数进行了高度泛化，它能表示0到任意个数、任意类型的参数。关于可变参数模板的原理和应用不是本文重点，不过通过本文中的例子也可充分了解可变参数模板是如何应用的。

熟悉Java或C#的同学应该都知道反射机制，很多有名的框架都用到了反射这种特性，简单的理解就是只根据类的名字（字符串）创建类的实例。 C++并没有直接从语言上提供反射机制给我们用，不过无所不能的C++可以通过一些trick来实现反射。Bwar也是在开发Nebula框架的时候需要用到反射机制，在网上参考了一些资料结合自己对C++11可变参数模板的理解实现了C++反射。

### C++11之前的模拟反射机制实现

Nebula框架是一个高性能事件驱动通用网络框架，框架本身无任何业务逻辑实现，却为快速实现业务提供了强大的功能、统一的接口。业务逻辑通过从Nebula的Actor类接口编译成so动态库，Nebula加载业务逻辑动态库实现各种功能Server，开发人员只需专注于业务逻辑代码编写，网络通信、定时器、数据序列化反序列化、采用何种通信协议等全部交由框架完成。

开发人员编写的业务逻辑类从Nebula的基类派生，但各业务逻辑派生类对Nebula来说是完全未知的，Nebula需要加载这些动态库并创建动态库中的类实例就需要用到反射机制。第一版的Nebula及其前身Starship框架以C++03标准开发，未知类名、未知参数个数、未知参数类型、更未知实参的情况下，Bwar没想到一个有效的加载动态库并创建类实例的方式。为此，将所有业务逻辑入口类的构造函数都设计成无参构造函数。

Bwar在2015年未找到比较好的实现，自己想了一种较为取巧的加载动态库并创建类实例的方法（这还不是反射机制，只是实现了可以或需要反射机制来实现的功能）。这个方法在Nebula的C++03版本中应用并实测通过，同时也是在一个稳定运行两年多的IM底层框架Starship中应用。下面给出这种实现方法的代码：

CmdHello.hpp:

```
#ifdef __cplusplus
extern "C" {
    #endif
    // @brief 创建函数声明
    // @note 插件代码编译成so后存放到plugin目录，框架加载动态库后调用create()创建插件类实例。
    neb::Cmd* create();
    #ifdef __cplusplus
}
#endif

namespace im
{

    class CmdHello: public neb::Cmd
    {
        public:
        CmdHello();
        virtual ~CmdHello();
        virtual bool AnyMessage();
    };

} /* namespace im */
```
CmdHello.cpp:

```
#include "CmdHello.hpp"

#ifdef __cplusplus
extern "C" {
#endif
    neb::Cmd* create()
    {
        neb::Cmd* pCmd = new im::CmdHello();
        return(pCmd);
    }
#ifdef __cplusplus
}
#endif

namespace im
{

    CmdHello::CmdHello()
    {
    }

    CmdHello::~CmdHello()
    {
    }

    bool CmdHello::AnyMessage()
    {
        std::cout << "CmdHello" << std::endl;
        return(true);
    }

}
```

实现的关键在于create()函数，虽然每个动态库都写了create()函数，但动态库加载时每个create()函数的地址是不一样的，通过不同地址调用到的函数也就是不一样的，所以可以创建不同的实例。下面给出动态库加载和调用create()函数，创建类实例的代码片段：

```
void* pHandle = NULL;
pHandle = dlopen(strSoPath.c_str(), RTLD_NOW);
char* dlsym_error = dlerror();
if (dlsym_error)
{
    LOG4_FATAL("cannot load dynamic lib %s!" , dlsym_error);
    if (pHandle != NULL)
    {
        dlclose(pHandle);
    }
    return(pSo);
}
CreateCmd* pCreateCmd = (CreateCmd*)dlsym(pHandle, strSymbol.c_str());
dlsym_error = dlerror();
if (dlsym_error)
{
    LOG4_FATAL("dlsym error %s!" , dlsym_error);
    dlclose(pHandle);
    return(pSo);
}
Cmd* pCmd = pCreateCmd();
```
对应这动态库加载代码片段的配置文件如下：

```
{"cmd":10001, "so_path":"plugins/CmdHello.so", "entrance_symbol":"create", "load":false, "version":1}
```
这些代码实现达到了加载动态库并创建框架未知类实例的目的。不过没有反射机制那么灵活，用起来也稍显麻烦，开发人员写好业务逻辑类之后还需要实现一个对应的全局create()函数。

### C++反射机制实现思考

Bwar曾经用C++模板封装过一个基于pthread的通用线程类。下面是这个线程模板类具有代表性的一个函数实现，对设计C++反射机制实现有一定的启发：

```
template <typename T>
void CThread<T>::StartRoutine(void* para)
{
    T* pT;
    pT = (T*) para;
    pT->Run();
}
```

与之相似，创建一个未知的类实例可以通过new T()的方式来实现，如果是带参数的构造函数，则可以new T(T1, T2)来实现。那么，通过类名创建类实例，建立"ClassName"与T的对应关系，或者建立"ClassName"与包含了new T()语句的函数的对应关系即可实现无参构造类的反射机制。考虑到new T(T1, T2)的参数个数和参数类型都未知，应用C++11的可变参数模板解决参数问题，即完成带参构造类的反射机制。

### Nebula网络框架中的C++反射机制实现

研究C++反射机制实现最重要是Nebula网络框架中有极其重要的应用，而Nebula框架在实现并应用了反射机制之后代码量精简了10%左右，同时易用性也有了很大的提高，再考虑到应用反射机制后给基于Nebula的业务逻辑开发带来的好处，可以说反射机制是Nebula框架仅次于以C++14标准重写的重大提升。

Nebula的Actor为事件（消息）处理者，所有业务逻辑均抽象成事件和事件处理，反射机制正是应用在Actor的动态创建上。Actor分为Cmd、Module、Step、Session四种不同类型。业务逻辑代码均通过从这四种不同类型时间处理者派生子类来实现，专注于业务逻辑实现，而无须关注业务逻辑之外的内容。Cmd和Module都是消息处理入库，业务开发人员定义了什么样的Cmd和Module对框架而言是未知的，因此这些Cmd和Module都配置在配置文件里，Nebula通过配置文件中的Cmd和Module的名称（字符串）完成它们的实例创建。通过反射机制动态创建Actor的关键代码如下：

Actor类：

```
class Actor: public std::enable_shared_from_this<Actor>
```

Actor创建工厂(注意看代码注释):
```
template<typename ...Targs>
class ActorFactory
{
    public:
    static ActorFactory* Instance()
    {
        if (nullptr == m_pActorFactory)
        {
            m_pActorFactory = new ActorFactory();
        }
        return(m_pActorFactory);
    }

    virtual ~ActorFactory(){};

    // 将“实例创建方法（DynamicCreator的CreateObject方法）”注册到ActorFactory，注册的同时赋予这个方法一个名字“类名”，后续可以通过“类名”获得该类的“实例创建方法”。这个实例创建方法实质上是个函数指针，在C++11里std::function的可读性比函数指针更好，所以用了std::function。
    bool Regist(const std::string& strTypeName, std::function<Actor*(Targs&&... args)> pFunc);

    // 传入“类名”和参数创建类实例，方法内部通过“类名”从m_mapCreateFunction获得了对应的“实例创建方法（DynamicCreator的CreateObject方法）”完成实例创建操作。
    Actor* Create(const std::string& strTypeName, Targs&&... args);

    private:
    ActorFactory(){};
    static ActorFactory<Targs...>* m_pActorFactory;
            std::unordered_map<std::string, std::function<Actor*(Targs&&...)> > m_mapCreateFunction;
};

template<typename ...Targs>
ActorFactory<Targs...>* ActorFactory<Targs...>::m_pActorFactory = nullptr;

template<typename ...Targs>
bool ActorFactory<Targs...>::Regist(const std::string& strTypeName, std::function<Actor*(Targs&&... args)> pFunc)
{
    if (nullptr == pFunc)
    {
        return (false);
    }

    bool bReg = m_mapCreateFunction.insert(   std::make_pair(strTypeName, pFunc)).second;
    return (bReg);
}

template<typename ...Targs>
Actor* ActorFactory<Targs...>::Create(const std::string& strTypeName, Targs&&... args)
{
    auto iter = m_mapCreateFunction.find(strTypeName);
    if (iter == m_mapCreateFunction.end())
    {
        return (nullptr);
    } else {
        return (iter->second(std::forward<Targs>(args)...));
    }
}
```
动态创建类（注意看代码注释）：

```
template<typename T, typename...Targs>
class DynamicCreator
{
public:
    struct Register
    {
        Register()
        {
            char* szDemangleName = nullptr;
            std::string strTypeName;
#ifdef __GUNC__
            szDemangleName = abi::__cxa_demangle(typeid(T).name(), nullptr, nullptr, nullptr);
#else
            // 注意：这里不同编译器typeid(T).name()返回的字符串不一样，需要针对编译器写对应的实现
            //in this format?:     szDemangleName =  typeid(T).name();
            szDemangleName = abi::__cxa_demangle(typeid(T).name(), nullptr, nullptr, nullptr);
#endif
            if (nullptr != szDemangleName)
            {
                strTypeName = szDemangleName;
                free(szDemangleName);
            }
            ActorFactory<Targs...>::Instance()->Regist(strTypeName, CreateObject);
        }
        inline void do_nothing()const { };
    };

    DynamicCreator()
    {
        m_oRegister.do_nothing();   // 这里的函数调用虽无实际内容，却是在调用动态创建函数前完成m_oRegister实例创建的关键
    }
    virtual ~DynamicCreator(){};

    // 动态创建实例的方法，所有Actor实例均通过此方法创建。这是个模板方法，实际上每个Actor的派生类都对应了自己的CreateObject方法。
    static T* CreateObject(Targs&&... args)
    {
        T* pT = nullptr;
        try
        {
            pT = new T(std::forward<Targs>(args)...);
        }
        catch(std::bad_alloc& e)
        {
            return(nullptr);
        }
        return(pT);
    }

private:
    static Register m_oRegister;
};

template<typename T, typename ...Targs>
typename DynamicCreator<T, Targs...>::Register DynamicCreator<T, Targs...>::m_oRegister;
```
上面ActorFactory和DynamicCreator就是C++反射机制的全部实现。要完成实例的动态创建还需要类定义必须满足（模板）要求。下面看一个可以动态创建实例的CmdHello类定义（注意看代码注释）：

```
// 类定义需要使用多重继承。
// 第一重继承neb::Cmd是CmdHello的实际基类（neb::Cmd为Actor的派生类，Actor是什么在本节开始的描述中有说明）；
// 第二重继承为通过类名动态创建实例的需要，与template<typename T, typename...Targs> class DynamicCreator定义对应着看就很容易明白第一个模板参数（CmdHello）为待动态创建的类名，其他参数为该类的构造函数参数。
// 如果参数为某个类型的指针或引用，作为模板参数时应指定到类型。比如： 参数类型const std::string&只需在neb::DynamicCreator的模板参数里填std::string。
class CmdHello: public neb::Cmd, public neb::DynamicCreator<CmdHello, int32>
{
public:
    CmdHello(int32 iCmd);
    virtual ~CmdHello();

    virtual bool Init();
    virtual bool AnyMessage(
                    std::shared_ptr<neb::SocketChannel> pChannel,
                    const MsgHead& oMsgHead,
                    const MsgBody& oMsgBody);
};
```

再看看上面的反射机制是怎么调用的:
```
template <typename ...Targs>
std::shared_ptr<Cmd> WorkerImpl::MakeSharedCmd(Actor* pCreator, const std::string& strCmdName, Targs... args)
{
    LOG4_TRACE("%s(CmdName \"%s\")", __FUNCTION__, strCmdName.c_str());
    Cmd* pCmd = dynamic_cast<Cmd*>(ActorFactory<Targs...>::Instance()->Create(strCmdName, std::forward<Targs>(args)...));
    if (nullptr == pCmd)
    {
        LOG4_ERROR("failed to make shared cmd \"%s\"", strCmdName.c_str());
        return(nullptr);
    }
    ...
}
```

MakeSharedCmd()方法的调用:
```
MakeSharedCmd(nullptr, oCmdConf["cmd"][i]("class"), iCmd);
```
至此通过C++可变参数模板实现C++反射机制实现已全部讲完，相信读到这里已经有了一定的理解，这是Nebula框架的核心功能之一，已有不少基于Nebula的应用实践，是可用于生产的C++反射实现。

这个C++反射机制的应用容易出错的地方是:
- 类定义class CmdHello: public neb::Cmd, public neb::DynamicCreator<CmdHello, int32>中的模板参数一定要与构造函数中的参数类型严格匹配（不明白的请再阅读一遍CmdHello类定义）。
- 调用创建方法的地方传入的实参类型必须与形参类型严格匹配。不能有隐式的类型转换，比如类构造函数的形参类型为unsigned int，调用ActorFactory<Targs...>::Instance()->Create()时传入的实参为int或short或unsigned short或enum都会导致ActorFactory无法找到对应的“实例创建方法”，从而导致不能通过类名正常创建实例。

注意以上两点，基本就不会有什么问题了。


### 一个可执行的例子

上面为了说明C++反射机制给出的代码全都来自Nebula框架。最后再提供一个可执行的例子加深理解。
DynamicCreate.cpp:

```
#include <string>
#include <iostream>
#include <typeinfo>
#include <memory>
#include <unordered_map>
#include <cxxabi.h>

namespace neb
{

class Actor
{
public:
    Actor(){std::cout << "Actor construct" << std::endl;}
    virtual ~Actor(){};
    virtual void Say()
    {
        std::cout << "Actor" << std::endl;
    }
};

template<typename ...Targs>
class ActorFactory
{
public:
    //typedef Actor* (*ActorCreateFunction)();
    //std::function< Actor*(Targs...args) > pp;

    static ActorFactory* Instance()
    {
        std::cout << "static ActorFactory* Instance()" << std::endl;
        if (nullptr == m_pActorFactory)
        {
            m_pActorFactory = new ActorFactory();
        }
        return(m_pActorFactory);
    }

    virtual ~ActorFactory(){};

    //Lambda: static std::string ReadTypeName(const char * name)

    //bool Regist(const std::string& strTypeName, ActorCreateFunction pFunc)
    //bool Regist(const std::string& strTypeName, std::function<Actor*()> pFunc)
    bool Regist(const std::string& strTypeName, std::function<Actor*(Targs&&... args)> pFunc)
    {
        std::cout << "bool ActorFactory::Regist(const std::string& strTypeName, std::function<Actor*(Targs... args)> pFunc)" << std::endl;
        if (nullptr == pFunc)
        {
            return(false);
        }
        std::string strRealTypeName = strTypeName;
        //[&strTypeName, &strRealTypeName]{int iPos = strTypeName.rfind(' '); strRealTypeName = std::move(strTypeName.substr(iPos+1, strTypeName.length() - (iPos + 1)));};

        bool bReg = m_mapCreateFunction.insert(std::make_pair(strRealTypeName, pFunc)).second;
        std::cout << "m_mapCreateFunction.size() =" << m_mapCreateFunction.size() << std::endl;
        return(bReg);
    }

    Actor* Create(const std::string& strTypeName, Targs&&... args)
    {
        std::cout << "Actor* ActorFactory::Create(const std::string& strTypeName, Targs... args)" << std::endl;
        auto iter = m_mapCreateFunction.find(strTypeName);
        if (iter == m_mapCreateFunction.end())
        {
            return(nullptr);
        }
        else
        {
            //return(iter->second());
            return(iter->second(std::forward<Targs>(args)...));
        }
    }

private:
    ActorFactory(){std::cout << "ActorFactory construct" << std::endl;};
    static ActorFactory<Targs...>* m_pActorFactory;   
    std::unordered_map<std::string, std::function<Actor*(Targs&&...)> > m_mapCreateFunction;
};

template<typename ...Targs>
ActorFactory<Targs...>* ActorFactory<Targs...>::m_pActorFactory = nullptr;

template<typename T, typename ...Targs>
class DynamicCreator
{
public:
    struct Register
    {
        Register()
        {
            std::cout << "DynamicCreator.Register construct" << std::endl;
            char* szDemangleName = nullptr;
            std::string strTypeName;
#ifdef __GUNC__
            szDemangleName = abi::__cxa_demangle(typeid(T).name(), nullptr, nullptr, nullptr);
#else
            //in this format?:     szDemangleName =  typeid(T).name();
            szDemangleName = abi::__cxa_demangle(typeid(T).name(), nullptr, nullptr, nullptr);
#endif
            if (nullptr != szDemangleName)
            {
                strTypeName = szDemangleName;
                free(szDemangleName);
            }

            ActorFactory<Targs...>::Instance()->Regist(strTypeName, CreateObject);
        }
        inline void do_nothing()const { };
    };
    DynamicCreator()
    {
        std::cout << "DynamicCreator construct" << std::endl;
        m_oRegister.do_nothing();
    }
    virtual ~DynamicCreator(){m_oRegister.do_nothing();};

    static T* CreateObject(Targs&&... args)
    {
        std::cout << "static Actor* DynamicCreator::CreateObject(Targs... args)" << std::endl;
        return new T(std::forward<Targs>(args)...);
    }

    virtual void Say()
    {
        std::cout << "DynamicCreator say" << std::endl;
    }
    static Register m_oRegister;
};

template<typename T, typename ...Targs>
typename DynamicCreator<T, Targs...>::Register DynamicCreator<T, Targs...>::m_oRegister;

class Cmd: public Actor, public DynamicCreator<Cmd>
{
public:
    Cmd(){std::cout << "Create Cmd " << std::endl;}
    virtual void Say()
    {
        std::cout << "I am Cmd" << std::endl;
    }
};

class Step: public Actor, DynamicCreator<Step, std::string, int>
{
public:
    Step(const std::string& strType, int iSeq){std::cout << "Create Step " << strType << " with seq " << iSeq << std::endl;}
    virtual void Say()
    {
        std::cout << "I am Step" << std::endl;
    }
};

class Worker
{
public:
    template<typename ...Targs>
    Actor* CreateActor(const std::string& strTypeName, Targs&&... args)
    {
        Actor* p = ActorFactory<Targs...>::Instance()->Create(strTypeName, std::forward<Targs>(args)...);
        return(p);
    }
};

}

int main()
{
    //Actor* p1 = ActorFactory<std::string, int>::Instance()->Create(std::string("Cmd"), std::string("neb::Cmd"), 1001);
    //Actor* p3 = ActorFactory<>::Instance()->Create(std::string("Cmd"));
    neb::Worker W;
    neb::Actor* p1 = W.CreateActor(std::string("neb::Cmd"));
    p1->Say();
    //std::cout << abi::__cxa_demangle(typeid(Worker).name(), nullptr, nullptr, nullptr) << std::endl;
    std::cout << "----------------------------------------------------------------------" << std::endl;
    neb::Actor* p2 = W.CreateActor(std::string("neb::Step"), std::string("neb::Step"), 1002);
    p2->Say();
    return(0);
}
```
Nebula框架是用C++14标准写的，在Makefile中有预编译选项，可以用C++11标准编译，但未完全支持C++11全部标准的编译器可能无法编译成功。实测g++ 4.8.5不支持可变参数模板，建议采用gcc 5.0以后的编译器，最好用gcc 6，Nebula用的是gcc6.4。


这里给出的例子DynamicCreate.cpp可以这样编译：

```
g++ -std=c++11 DynamicCreate.cpp -o DynamicCreate
```
程序执行结果如下:

```
DynamicCreator.Register construct
static ActorFactory* Instance()
ActorFactory construct
bool ActorFactory::Regist(const std::string& strTypeName, std::function<Actor*(Targs... args)> pFunc)
m_mapCreateFunction.size() =1
DynamicCreator.Register construct
static ActorFactory* Instance()
ActorFactory construct
bool ActorFactory::Regist(const std::string& strTypeName, std::function<Actor*(Targs... args)> pFunc)
m_mapCreateFunction.size() =1
static ActorFactory* Instance()
Actor* ActorFactory::Create(const std::string& strTypeName, Targs... args)
static Actor* DynamicCreator::CreateObject(Targs... args)
Actor construct
DynamicCreator construct
Create Cmd
I am Cmd
----------------------------------------------------------------------
static ActorFactory* Instance()
Actor* ActorFactory::Create(const std::string& strTypeName, Targs... args)
static Actor* DynamicCreator::CreateObject(Targs... args)
Actor construct
DynamicCreator construct
Create Step neb::Step with seq 1002
I am Step
```
完毕，周末花了6个小时才写完，找了个合适的时间发布出来。如果觉得这篇文章对你有用，如果觉得Nebula还可以，麻烦到GitHub上给个star，感谢。 Nebula不仅是一个框架，还提供了一系列基于这个框架的应用，目标是打造一个高性能分布式服务集群解决方案。

### 参考资料

[C++实现反射(根据类名动态创建对象)](https://blog.csdn.net/heyuhang112/article/details/51729435)

[泛化之美--C++11可变模版参数的妙用](https://www.cnblogs.com/qicosmos/p/4325949.html)

[C++ 标准库之typeid](https://blog.csdn.net/xuqingict/article/details/25003713)
