---
layout:     post
title:      "状态模式"
subtitle:   "基础介绍"
date:       2018-03-28 13:27:00
author:     "邹盛富"
header-img: "img/state_model.png"
tags:
    - 设计模式
---

### 使用意图
使行为自动适应状态的改变，去掉if或者case语句

### 结构图

![结构图](https://github.com/Stephan14/Stephan14.github.io/blob/master/img/struct.png)

### 使用场景
- 对象收到其他对象的请求时，根据自身的不同状态做出不同的反应
- 一个操作中含有大量的条件分支语句，并且这些分支依赖于状态

### 优点

- 通过增加State的子类可以容易的增加新的状态和转化
- 状态转换的时候，Context类中只需要重新绑定一个State变量，无须重新赋值，避免内部状态不一致
- State对象可以被共享

### 缺点

- 增加了子类的数目，类之间比较混乱

### 实现
- 由谁定义状态的状态的转换：
  - Context类中定义后继状态以及何时进行转换
  - 由State子类定义后继状态以及何时进行转换
- 创建和销毁State子类对象：
  - 状态在运行时是不可知的并且不经常改变状态时，对象使用时创建不使用时销毁
  - 状态对象含有大量信息且状态变换频繁时，在运行前一次创建所有对象，运行时不销毁，运行结束后销毁
- Context类中必须含有一个changeState的改变状态的方法供State类方法调用

### 与其他模式关系
- State子类通常使用单例模式
- 与策略模式相比：状态模式相互转换不是用户指定的，而是自动转换，相对安全

### 示例代码

```
//  
//  TcpState.h  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#ifndef TcpState_h  
#define TcpState_h  

#include <iostream>  
#include <string>  
class TcpConnection;  
//#include "TcpConnection.h"  
using namespace std;  
class TcpState  
{  
public:  
    TcpState(){  
        cout<<"TcpState初始化"<<endl;  
        cout<<instance<<endl;  
    }  
    virtual ~TcpState(){}  
    /*把子类的所有行为封装到接口中,每个函数都有一个TcpConnection作为参数从而访问TcpConnection中的数据和改变连接的状态*/  
    virtual void transmit(TcpConnection* con,string message){  
        cout<<message<<endl;}  
    virtual void activeOpen(TcpConnection* con){}  
    virtual void passiveOpen(TcpConnection* con){}  
    virtual void close(TcpConnection* con){}  
    virtual void syschronize(TcpConnection* con){}  
    virtual void acknowledge(TcpConnection* con){}  
    virtual void send(TcpConnection* con){}  
    static void release(){ std::cout<<"TcpState"<<endl;}  
protected:  
    /*可有可无，其本质就是调用Context方法改变其状态，可再其方法中实现*/  
    void changeState(TcpConnection* con, TcpState* state);  
    static TcpState* instance;  
};  

#endif /* TcpState_h */  
//  
//  TcpState.cpp  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#include "TcpState.h"  
#include "TcpConnection.h"  

TcpState* TcpState::instance = 0;  
void TcpState::changeState(TcpConnection *con, TcpState *state)  
{  
    con->changeState(state);  
}  

//  
//  TcpListen.h  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#ifndef TcpListen_h  
#define TcpListen_h  

#include <iostream>  
#include "TcpState.h"  

class TcpListen:public TcpState  
{  
public:  
    static TcpListen* getInstace()  
    {  
        if (instance == nullptr) {  
            return new TcpListen;  
        }  
        return dynamic_cast<TcpListen*>(instance);  
    }  

    static void release()  
    {  
        if (instance != nullptr) {  
            delete instance;  
            instance = nullptr;  
        }  
    }  

    virtual void send(TcpConnection *con);  
private:  
    TcpListen(){}  
    TcpListen(const TcpListen& );  
    TcpListen& operator= (const TcpListen&);  
    virtual ~TcpListen(){}  

    //static TcpState* instance;  
};  

#endif /* TcpListen_h */  

//  
//  TcpListen.cpp  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#include "TcpListen.h"  
#include "TcpEstablished.h"  
using namespace std;  
//TcpState* TcpListen::instance = 0;  
void TcpListen::send(TcpConnection *con)  
{  
    cout<<"TcpListen::send"<<endl;  
    changeState(con, TcpEstablished::getInstance());  
}  

//  
//  TcpEstablished.h  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#ifndef TcpEstablished_h  
#define TcpEstablished_h  

#include <iostream>  
#include "TcpState.h"  

class TcpEstablished:public TcpState  
{  
public:  
    /*单例设计模式*/  
    static TcpEstablished* getInstance(){  
        if (instance == nullptr) {  
            return new TcpEstablished;  
        }  
        return dynamic_cast<TcpEstablished*>(instance);  
    }  

    static void release(){  
        if (instance != nullptr) {  
            delete instance;  
            instance = nullptr;  
        }  
    }  

    virtual void transmit(TcpConnection *con, string message);  
    virtual void close(TcpConnection *con);  
private:  
    TcpEstablished(){}  
    virtual ~TcpEstablished(){}  
    TcpEstablished(const TcpEstablished& );  
    TcpEstablished& operator=(const TcpEstablished& );  

    //static TcpState* instance;  
};  

#endif /* TcpEstablished_h */  

//  
//  TcpEstablished.cpp  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#include "TcpEstablished.h"  
#include "TcpListen.h"  

using namespace std;  
//TcpState* TcpEstablished::instance = 0;  
void TcpEstablished::transmit(TcpConnection *con, string message)  
{  
    cout<<"TcpEstablished::transmit"<<message<<endl;  
    changeState(con, TcpListen::getInstace());  

}  

void TcpEstablished::close(TcpConnection *con)  
{  
    cout<<"TcpEstablished::close"<<endl;  
    /*具体定义转换成哪种状态，调用TcpConnection类的changeState统一执行*/  
    changeState(con, TcpListen::getInstace());  
}

//  
//  TcpClosed.h  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#ifndef TcpClosed_h  
#define TcpClosed_h  

#include <iostream>  
#include "TcpState.h"  

class TcpClosed:public TcpState  
{  
public:  
    static TcpClosed* getInstance(){  
        if (instance == nullptr) {  
            return new TcpClosed;  
        }  
        return dynamic_cast<TcpClosed*>(instance);  
    }  

    static void release(){  
        if (instance != nullptr) {  
            delete instance;  
            instance = nullptr;  
        }  
    }  

    virtual void activeOpen(TcpConnection *con);  
    virtual void passiveOpen(TcpConnection* con);  
private:  
    TcpClosed(){}  
    virtual ~TcpClosed(){}  
    TcpClosed(const TcpClosed&);  
    TcpClosed& operator= (const TcpClosed&);  

};  

#endif /* TcpClosed_h */  

//  
//  TcpClosed.cpp  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#include "TcpClosed.h"  
#include "TcpEstablished.h"  
#include "TcpListen.h"  
using namespace std;  

void TcpClosed::activeOpen(TcpConnection *con)  
{  
    cout<<"TcpClosed::activeOpen"<<endl;  
    changeState(con, TcpEstablished::getInstance());  
}  

void TcpClosed::passiveOpen(TcpConnection *con)  
{  
    cout<<"TcpClosed::passiveOpen"<<endl;  
    changeState(con, TcpListen::getInstace());  
}

//  
//  TcpConnection.h  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#ifndef TcpConnection_h  
#define TcpConnection_h  

#include <iostream>  
#include "TcpState.h"  
#include "TcpListen.h"  
#include "TcpEstablished.h"  
#include "TcpClosed.h"  

using namespace std;  

class TcpConnection  
{  
public:  
    TcpConnection(){ state = TcpClosed::getInstance(); }  
    virtual ~TcpConnection(){ /*state->release();*/}  

    void activeOpen(){ state->activeOpen(this); }  
    void passiveOpen(){ state->passiveOpen(this); }  
    void close(){ state->close(this); }  
    void send(){ state->send(this); }  
    void acknowledge(){ state->acknowledge(this); }  
    void synchronize(){ state->syschronize(this); }  
    void transmit(string message){ state->transmit(this, message); }  
private:  
    /*必须要有changeState方法供状态类来调用改变状态*/  
    void changeState(TcpState *state){this->state = state;}  
    friend class TcpState;  
private:  
    TcpState* state;  

};  

#endif /* TcpConnection_h */  

//  
//  TcpConnection.cpp  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#include "TcpConnection.h"  


//  
//  main.cpp  
//  textD  
//  
//  Created by md101 on 15/10/19.  
//  Copyright © 2015年 md101. All rights reserved.  
//  

#include <iostream>  
#include "TcpConnection.h"  

using namespace std;  

int main(int argc, const char * argv[]) {  

    TcpConnection *con = new TcpConnection();  
    con->activeOpen();  
    con->transmit("xgdsgdfsg");  
    con->send();  
    TcpClosed::release();  
    TcpEstablished::release();  
    TcpListen::release();  
    delete con;  

    return 0;  
}
```
