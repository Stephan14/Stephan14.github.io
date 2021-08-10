---
layout:     post
title:      "C++类型转换"
subtitle:   "C++四种类型转换区别"
date:       2021-08-03 09:42:30
author:     "zoushengfu"
header-img: "img/ocean-6519233.jpeg"
tags:
    - C/C++
---

## 背景
- C 风格（C-style）强制转型如下：
(T) expression // cast expression to be of type T
- 函数风格（Function-style）强制转型使用这样的语法：
T(expression) // cast expression to be of type T

这两种形式之间没有本质上的不同，它纯粹就是一个把括号放在哪的问题。我把这两种形式称为旧风格（old-style）的强制转型。

**有时用c风格的转换是不合适的，因为它可以在任意类型之间转换，比如你可以把一个指向const对象的指针转换成指向非 const对象的指针，把一个指向基类对象的指针转换成指向一个派生类对象的指针，这两种转换之间的差别是巨大的，但是传统的c语言风格的类型转换没有区分这些**

## C++的四种强制转型形式
C++ 同时提供了四种新的强制转型形式（通常称为新风格的或 C++ 风格的强制转型）
- const_cast(expression)   
- dynamic_cast(expression)
- reinterpret_cast(expression)
- static_cast(expression)

### static_cast转换
可以实现C++中 内置基本数据类型之间的相互转换，enum、struct、 int、char、float等，不能进行无关类型(如非基类和子类)指针之间的转换。**如果涉及到类的话，static_cast只能在有 相互联系的类型 中进行相互转换,不一定包含虚函数。**

```
class A
{};
class B:public A
{};
class C
{};

int main()
{
    A* a=new A;
    B* b;
    C* c;
    b=static_cast<B>(a);  // 编译不会报错, B类继承A类
    c=static_cast<B>(a);  // 编译报错, C类与A类没有任何关系
    return 1;
}
```

### const_cast转换
const_cast操作不能在不同的种类间转换。相反，它仅仅把一个它作用的表达式转换成常量。它可以使一个本来不是const类型的数据转换成const类型的，或者把const属性去掉。

```
const string& shorter(const string& s1, const string& s2) {
	return s1.size() <= s2.size() ? s1 : s2;
}

string& shorter(string& s1, string& s2) {
	//重载调用到上一个函数，它已经写好了比较的逻辑
	auto &r = shorter(const_cast<const string&>(s1), const_cast<const string&>(s2));
	//auto等号右边为引用，类型会忽略掉引用
	return const_cast<string&>(r);
}
```

### reinterpret_cast转换

interpret是解释的意思，reinterpret即为重新解释，此标识符的意思即为数据的二进制形式重新解释，但是不改变其值。有着和C风格的强制转换同样的能力。它可以转化任何内置的数据类型为其他任何的数据类型，也可以转化任何指针类型为其他的类型。它甚至可以转化内置的数据类型为指针，无须考虑类型安全或者常量的情形。 不到万不得已绝对不用。

### dynamic_cast转换
- 其他三种都是编译时完成的，dynamic_cast是 运行时处理的，运行时要进行类型检查
- 不能用于内置的基本数据类型的强制转换
- dynamic_cast转换如果成功的话返回的是指向类的指针或引用，**转换失败的话则会返回NULL**
- 使用dynamic_cast进行转换的，基类中一定要有虚函数，否则编译不通过

        需要检测有虚函数的原因：类中存在虚函数，就说明它有想要让基类指针或引用指向派生类对象的情况，此时转换才有意义。这是由于运行时类型检查需要运行时类型信息，而这个信息存储在类的虚函数表（关于虚函数表的概念，详细可见<Inside c++ object model>）中，只有定义了虚函数的类才有虚函数表。


- 在类的转换时，在类层次间进行上行转换时，dynamic_cast和static_cast的效果是一样的。在进行下行转换时，dynamic_cast具有类型检查的功能，比   static_cast更安全。向上转换即为指向子类对象的向下转换，即将父类指针转化子类指针。向下转换的成功与否还与将要转换的类型有关，即要转换的指针指向的对象的实际类型与转换以后的对象类型一定要相同，否则转换失败。

```
#include<iostream>
#include<cstring>
using namespace std;
class A
{
   public:
   virtual void f()
   {
       cout<<"hello"<<endl;
       };
};

class B:public A
{
    public:
    void f()
    {
        cout<<"hello2"<<endl;
        };

};

class C
{
  void pp()
  {
      return;
  }
};

int fun()
{
    return 1;
}
int main()
{
    A* a1=new B;//a1是A类型的指针指向一个B类型的对象
    A* a2=new A;//a2是A类型的指针指向一个A类型的对象
    B* b;
    C* c;
    b=dynamic_cast<B*>(a1);//结果为not null，向下转换成功，a1之前指向的就是B类型的对象，所以可以转换成B类型的指针。
    if(b==NULL)
    {
        cout<<"null"<<endl;
    }
    else
    {
        cout<<"not null"<<endl;
    }
    b=dynamic_cast<B*>(a2);//结果为null，向下转换失败
    if(b==NULL)
    {
        cout<<"null"<<endl;
    }
    else
    {
        cout<<"not null"<<endl;
    }
    c=dynamic_cast<C*>(a);//结果为null，向下转换失败
    if(c==NULL)
    {
        cout<<"null"<<endl;
    }
    else
    {
        cout<<"not null"<<endl;
    }
    delete(a);
    return 0;
}

```

## 隐式转换
隐式转换为向上转换，即将派生类对象赋值给基类，C++允许向上转型，即将派生类的对象赋值给基类的对象是可以的，其只不过是将派生类中基类的部分直接赋给基类的对象，这称为向上转型（此处的“上”指的是基类），例如：

```
class Base{ };
class Derived : public base{ };
Base* Bptr;
Derived* Dptr;
Bptr = Dptr; //编译正确，允许隐式向上类型转换
Dptr = Bptr；//编译错误，C++不允许隐式的向下转型；
```

## 向下转换
正如上面所述，类层次间的向下转型是不能通过隐式转换完成的。此时要向达到这种转换，可以借助static_cast 或者dynamic_cast。

### 通过static_cast完成向下转型

```
class Base{ };
class Derived : public base{ };
Base* B;
Derived* D;
D = static_cast<Drived*>(B); //正确，通过使用static_cast向下转型
```

**对于static_cast的使用，当且仅当类型之间可隐式转化时，static_cast的转化才是合法的。有一个例外，那就是类层次间的向下转型，static_cast可以完成类层次间的向下转型，但是向下转型无法通过隐式转换完成！**

### 通过dynamic_cast完成向下转型

和static_cast不同，dynamic_cast涉及运行时的类型检查。如果向下转型是安全的（也就是说，如果基类指针或者引用确实指向一个派生类的对象），这个运算符会传回转型过的指针。如果downcast不安全（即基类指针或者引用没有指向一个派生类的对象），这个运算符会传回空指针。

```
class Base{
public：
    virtual void fun(){}
 };
class Drived : public base{
public:
    int i;
 };
Base *Bptr = new D；//语句0
Derived *Dptr1 = static_cast<Derived *>(Bptr); //语句1；
Derived *Dptr2 = dynamic_cast<Derived *>(Bptr); //语句2；
```
此时语句1和语句2都是安全的，因为此时Bptr确实是指向的派生类，虽然其类型被声明为Base*，但是其实际指向的内容确确实实是Drived对象，所以语句1和2都是安全的，Dptr1和Dptr2可以尽情访问Drived类中的成员，绝对不会出问题。
但是此时如果将语句0改为这样：
```
Base *Bptr = new B；
```
那语句1就不安全了，例如访问Drived类的成员变量i的值时，将得到一个垃圾值。（延后了错误的发现）。语句2使得Dptr2得到的是一个空指针，对空指针进行操作，将发生异常，从而能够尽早的发现错误，这也是为什么说dynamic_cast更安全的原因。

### 多继承时的向下转型
```
class Base1{
    virtual void f1(){}
}；
class Base2{
    virtual void f2(){}
};
class Derived: public Base1, public Base2{
    void f1(){}
    void f2(){}
};
Base1 *pD = new Derived;
Derived *pD1  = dynamic_cast<Derived*>(pD);  //正确，原因和前面类似
Derived *pD2  = static_cast<Derived*>(pD);  //正确，原因和前面类似
Base2 *pB1  = dynamic_cast<Base2*>(pD);    //语句1
Base2 *pB2  = static_cast<Base2*>(pD);    //语句2
```
此时的语句1,将pD的类型转化为Base2\*，即使得pB1指向Drived对象的Base2子对象，为什么能达到这种转化？因为dynamic_cast是运行时才决定真正的类型，在运行时发现虽然此时pD的类型是Base1\*，但是实际指向的是Derived类型的对象，那么就可以通过调整指针，来达到pB1指向Derived 对象的Base2子对象的目的；但是语句2就不行了，其使用的是static_cast，它不涉及运行时的类型检查，对于它来讲，pD的类型是Base1\*，Base1和Base2没有任何关系，那就会出现编译错误了。

    error: invalid static_cast from type ‘Base1*’ to type ‘Base2*’

总结：对于多种继承，如果pD真的是指向Derived，使用static_cast和dynamic_cast都可以转化为Derived，但是如果要转化为Base1的兄弟类Base2，必须使用dynamic_cast，使用static_cast不能通过编译。
ps：因为Derived\*和Base1\*和Base2\*之间存在隐式转化，可以将语句2修改为：
```
Base2 *pB2 = static_cast<Base2*>(static_cast<Derived*>(pD));
```
这样就可以完成转换！
