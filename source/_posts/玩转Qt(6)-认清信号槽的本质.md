---
title: 玩转Qt(6)-认清信号槽的本质
photos: /images/Qt4/1.jpg
tags:
  - Qt
  - QtCreator
  - Qt实用技能
categories: 玩转Qt
abbrlink: 64079
date: 2019-07-23 18:44:23
---

- [简介](#%e7%ae%80%e4%bb%8b)
- [猫和老鼠的故事](#%e7%8c%ab%e5%92%8c%e8%80%81%e9%bc%a0%e7%9a%84%e6%95%85%e4%ba%8b)
- [对象之间的通信机制](#%e5%af%b9%e8%b1%a1%e4%b9%8b%e9%97%b4%e7%9a%84%e9%80%9a%e4%bf%a1%e6%9c%ba%e5%88%b6)
  - [尝试一：直接调用](#%e5%b0%9d%e8%af%95%e4%b8%80%e7%9b%b4%e6%8e%a5%e8%b0%83%e7%94%a8)
  - [尝试二：回调函数+映射表](#%e5%b0%9d%e8%af%95%e4%ba%8c%e5%9b%9e%e8%b0%83%e5%87%bd%e6%95%b0%e6%98%a0%e5%b0%84%e8%a1%a8)
- [观察者模式](#%e8%a7%82%e5%af%9f%e8%80%85%e6%a8%a1%e5%bc%8f)
- [Qt的信号-槽](#qt%e7%9a%84%e4%bf%a1%e5%8f%b7-%e6%a7%bd)
  - [信号-槽简介](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e7%ae%80%e4%bb%8b)
  - [信号-槽分两种](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e5%88%86%e4%b8%a4%e7%a7%8d)
  - [信号-槽的实现 元对象编译器moc](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e7%9a%84%e5%ae%9e%e7%8e%b0-%e5%85%83%e5%af%b9%e8%b1%a1%e7%bc%96%e8%af%91%e5%99%a8moc)
  - [moc的本质-反射](#moc%e7%9a%84%e6%9c%ac%e8%b4%a8-%e5%8f%8d%e5%b0%84)
- [参考文献](#%e5%8f%82%e8%80%83%e6%96%87%e7%8c%ae)


## 简介

这次讨论Qt信号-槽相关的知识点。

信号-槽是Qt框架中最核心的机制，也是每个Qt开发者必须掌握的技能。

网络上有很多介绍信号-槽的文章，也可以参考。

涛哥的专栏是《Qt进阶之路》，如果连信号-槽的文章都没有，将是没有灵魂的。

所以这次涛哥就由浅到深地说一说信号-槽。

## 猫和老鼠的故事

如果一上来就讲一大堆概念和定义，读者很容易读睡着。所以涛哥从一个故事/场景开始说起。

涛哥小时候喜欢看动画片《猫和老鼠》, 里面有汤姆猫(Tom)和杰瑞鼠(Jerry)斗智斗勇的故事。。。

![预览](/images/Qt4/1.jpg)

现在做个简单的设定：Tom有个技能叫"喵"，就是发出猫叫，而正在偷吃东西的Jerry,听见猫叫声就会逃跑。

![预览](/images/Qt4/TomCat.png)

![预览](/images/Qt4/jerry.jpg)

我们尝试用C++面向对象的思想，描述这个设定。

先是定义Tom和Jerry两种对象
```C++
//Tom的定义

class Tom
{
public:
    //猫叫
    void Miaow() 
    {
        cout << "喵!" << endl;
    }
    //省略其它
    ... 
};
//Jerry的定义
class Jerry
{
public:
    //逃跑
    void RunAway()
    {
        cout << "那只猫又来了，快溜！" << endl;
    }
    //省略其它
    ... 
};
```
接下来模拟场景
```C++
int main(int argc, char *argv[])
{
    //实例化tom
    Tom tom;

    //实例化jerry
    Jerry jerry;

    //tom发出叫声
    tom.Miaow();

    //jerry逃跑
    jerry.RunAway();

    return 0;
}

```
这个场景看起来很简单，tom发出叫声之后手动调用了jerry的逃跑。

我们再看几种稍微复杂的场景:

场景一:

假如jerry逃跑后过段时间，又回来偷吃东西。Tom再次发出叫声，jerry再次逃跑。。。

这个场景要重复几十次。我们能否实现，只要tom的Miaow被调用了，jerry的RunAway就自动被调用，而不是每次都手动调用?

场景二:

假如jerry是藏在“厨房的柜子里的米袋子后面”，无法直接发现它(不能直接获取到jerry对象，并调用它的函数)。

这种情况下，该怎么建立 “猫叫-老鼠逃跑” 的模型？

场景三：

假如有多只jerry，一只tom发出叫声时，所有jerry都逃跑。这种模型该怎么建立？

假如有多只tom，任意一只发出叫声时，所有jerry都逃跑。这种模型又该怎么建立？

场景四：

假如不知道猫的确切品种或者名字，也不知道老鼠的品种或者名字，只要 猫 这种动物发出叫声，老鼠 这种动物就要逃跑。

这样的模型又该如何建立?

...

 还有很多场景，就不赘述了。

## 对象之间的通信机制

这里概括一下要实现的功能：

要提供一种对象之间的通信机制。这种机制，要能够给两个不同对象中的函数建立映射关系，前者被调用时后者也能被自动调用。

再深入一些，两个对象都互相不知道对方的存在，仍然可以建立联系。甚至一对一的映射可以扩展到多对多，具体对象之间的映射可以扩展到抽象概念之间。

### 尝试一：直接调用

应该会有人说， Miaow()的函数中直接调用RunAway()不就行了？

明显场景二就把这种方案pass掉了。

直接调用的问题是，猫要知道老鼠有个函数/接口叫逃跑，然后主动调用了它。

这就好比Tom叫了一声，然后Tom主动拧着Jerry的腿让它跑。这样是不合理的。(Jerry表示一脸懵逼!)

真实的逻辑是，猫的叫声在空气/介质中传播，传到了老鼠的耳朵里，老鼠就逃跑了。猫和老鼠互相都没看见呢。

### 尝试二：回调函数+映射表

似乎是可行的。

稍微思考一下，我们要做这两件事情：

1 把RunAway函数取出来存储在某个地方

2 建立Miaow函数和RunAway的映射关系，能够在前者被调用时，自动调用后者。

RunAway函数可以用 函数指针|成员函数指针 或者C++11-function 来存储，都可以称作 "回调函数"。

(下面的代码以C++11 function的写法为主，函数指针的写法稍微复杂一些，本质一样)

我们先用一个简单的Map来存储映射关系, 就用一个字符串作为映射关系的名字

```c++
std::map<std::string, std::function<void()>> callbackMap;
```
我们还要实现 "建立映射关系" 和 "调用"功能，所以这里封装一个Connections类
```C++
class Connections 
{
public:
    //按名称“建立映射关系”
    void connect(const std::string &name, const std::function<void()> &callback) 
    {
        m_callbackMap[name] = callback;
    }
    //按名称“调用”
    void invok(const std::string &name)
    {
        auto it = m_callbackMap.find(name);
        //迭代器判断
        if (it != m_callbackMap.end()) {
            //迭代器有效的情况，直接调用
            it->second();
        }
    }
private:
    std::map<std::string, std::function<void()>> m_callbackMap;
};
```
那么这个映射关系存储在哪里呢? 显然是一个Tom和Jerry共有的"上下文环境"中。

我们用一个全局变量来表示，这样就可以简单地模拟了：
```c++
//全局共享的Connections。
static Connections s_connections;

//Tom的定义
class Tom
{
public:
    //猫叫
    void Miaow() 
    {
        cout << "喵!" << endl;
        //调用一下名字为mouse的回调
        s_connections.invok("mouse");
    }
    //省略其它
    ... 
};
//Jerry的定义
class Jerry
{
public:
    Jerry() 
    {
        //构造函数中，建立映射关系。std::bind属于基本用法。
        s_connections.connect("mouse", std::bind(&Jerry::RunAway, this));
    }
    //逃跑
    void RunAway()
    {
        cout << "那只猫又来了，快溜！" << endl;
    }
    //省略其它
    ... 
};
int main(int argc, char *argv[])
{
    //模拟嵌套层级很深的场景，外部不能直接访问到tom
    struct A {
        struct B {
            struct C {
                private:
                    //Tom在很深的结构中
                    Tom tom;
                public:
                    void MiaoMiaoMiao() 
                    {
                        tom.Miaow();
                    }
            }c;
            void MiaoMiao() 
            {
                c.MiaoMiaoMiao();
            }
        }b;
        void Miao() 
        {
            b.MiaoMiao();
        }
    }a;
    //模拟嵌套层级很深的场景，外部不能直接访问到jerry
    struct D {
        struct E {
            struct F {
                private:
                    //jerry在很深的结构中
                    Jerry jerry;
            }f;
        }e;
    }d;

    //A间接调用tom的MiaoW，发出猫叫声
    a.Miao();
    
    return 0;
}

```
看一下运行结果：

![预览](/images/Qt4/run.png)

RunAway没有被直接调用，而是被自动触发。

分析：这里是以"mouse"这个字符串作为连接tom和jerry的关键。这只是一种简单、粗糙的示例实现。

## 观察者模式

在GOF四人帮的书籍《设计模式》中，有一种观察者模式，可以比较优雅地实现同样的功能。

(顺便说一下，GOF总结的设计模式一共有23种，涛哥曾经用C++11实现了全套的，github地址是:https://github.com/jaredtao/DesignPattern)

初级的观察者模式，涛哥就不重复了。这里涛哥用C++11搭配一点模板技巧，实现一个更加通用的观察者模式。

也可以叫发布-订阅模式。
```C++
//Subject.hpp
#pragma once
#include <vector>
#include <algorithm>

//Subject 事件或消息的主体。模板参数为观察者类型
template<typename ObserverType>
class Subject {
public:
    //订阅
    void subscibe(ObserverType *obs)
    {
        auto itor = std::find(m_observerList.begin(), m_observerList.end(), obs);
        if (m_observerList.end() == itor) {
            m_observerList.push_back(obs);
        }
    }
    //取消订阅
    void unSubscibe(ObserverType *obs)
    {
        m_observerList.erase(std::remove(m_observerList.begin(), m_observerList.end(), obs));
    }
    //发布。这里的模板参数为函数类型。
    template <typename FuncType>
    void publish(FuncType func)
    {
        for (auto obs: m_observerList)
        {
            //调用回调函数，将obs作为第一个参数传递
            func(obs);
        }
    }
private:
    std::vector<ObserverType *> m_observerList;
};
```

```C++
//main.cpp
#include "Subject.hpp"
#include <functional>
#include <iostream>

using std::cout;
using std::endl;

//CatObserver 接口 猫的观察者
class CatObserver {
public:
    //猫叫事件
    virtual void onMiaow() = 0;
public:
    virtual ~CatObserver() {}
};

//Tom 继承于Subject模板类，模板参数为CatObserver。这样Tom就拥有了订阅、发布的功能。
class Tom : public Subject<CatObserver>
{
public:
    void miaoW()
    {
        cout << "喵!" << endl;
        //发布"猫叫"。
        //这里取CatObserver类的成员函数指针onMiaow。而成员函数指针调用时，要传递一个对象的this指针才行的。
        //所以用std::bind 和 std::placeholders::_1将第一个参数 绑定为 函数被调用时的第一个参数，也就是前面Subject::publish中的obs
        publish(std::bind(&CatObserver::onMiaow, std::placeholders::_1));
    }
};
//Jerry 继承于 CatObserver
class Jerry: public CatObserver
{
public:
    //重写“猫叫事件”
    void onMiaow() override
    {
        //发生 “猫叫”时 调用 逃跑
        RunAway();
    }
    void RunAway()
    {
        cout << "那只猫又来了，快溜！" << endl;
    }
};
int main(int argc, char *argv[])
{
    Tom tom;
    Jerry jerry;

    //拿jerry去订阅Tom的 猫叫事件
    tom.subscibe(&jerry);

    tom.miaoW();
    return 0;
}
```
任意类只要继承Subject模板类，提供观察者参数，就拥有了发布-订阅功能。

## Qt的信号-槽

### 信号-槽简介

信号-槽 是Qt自定义的一种通信机制，它不同于标准C/C++ 语言。

信号-槽的使用方法，是在普通的函数声明之前，加上signal、slot标记，然后通过connect函数把信号与槽 连接起来。

后续只要调用 信号函数,就可以触发连接好的信号或槽函数。

![预览](/images/Qt4/signals-slot.png)

连接的时候，前面的是发送者，后面的是接收者。信号与信号也可以连接，这种情况把接收者信号看做槽即可。

### 信号-槽分两种

信号-槽要分成两种来看待，一种是同一个线程内的信号-槽，另一种是跨线程的信号-槽。

同一个线程内的信号-槽，就相当于函数调用，和前面的观察者模式相似，只不过信号-槽稍微有些性能损耗(这个后面细说)。

跨线程的信号-槽，在信号触发时，发送者线程将槽函数的调用转化成了一次“调用事件”，放入事件循环中。

接收者线程执行到下一次事件处理时，处理“调用事件”，调用相应的函数。

(关于事件循环，可以参考专栏上一篇文章《Qt实用技能3-理解事件循环》)

### 信号-槽的实现 元对象编译器moc

信号-槽的实现，借助一个工具：元对象编译器MOC(Meta Object Compiler)。

这个工具被集成在了Qt的编译工具链qmake中，在开始编译Qt工程时，会先去执行MOC，从代码中

解析signals、slot、emit等等这些标准C/C++不存在的关键字，以及处理Q_OBJECT、Q_PROPERTY、

Q_INVOKABLE等相关的宏，生成一个moc_xxx.cpp的C++文件。(使用黑魔法来变现语法糖)

比如信号函数只要声明、不需要自己写实现，就是在这个moc_xxx.cpp文件中，自动生成的。

MOC之后就是常规的C/C++编译、链接流程了。

### moc的本质-反射

MOC的本质，其实是一个反射器。标准C++没有反射功能(将来会有)，所以Qt用moc实现了反射功能。

什么叫反射呢？ 简单来说，就是运行过程中，获取对象的构造函数、成员函数、成员变量。

举个例子来说明，有下面这样一个类声明：

```C++
class Tom {
public:
    Tom() {}
    const std::string & getName() const
    {
        return m_name;
    }
    void setName(const std::string &name) 
    {
        m_name = name;
    }
private:
    std::string m_name;
};
```
类的使用者,看不到类的声明,头文件都拿不到,不能直接调用类的构造函数、成员函数。

从配置文件/网络拿到了一段字符串“Tom”，就要创建一个Tom类的对象实例。

然后又拿到一段“setName”的字符串，就要去调用Tom的setName函数。

面对这种需求，就需要把Tom类的构造函数、成员函数等信息存储起来，还要能够被调用到。

这些信息就是 “元信息”，使用者通过“元信息”就可以“使用这个类”。这便是反射了。

设计模式中的“工厂模式”，就是一个典型的反射案例。不过工厂模式只解决了构造函数的调用，没有成员函数、成员变量等信息。

反射包括 编译期静态反射 和 运行期动态反射。。。

文章有点长了，这次先到这里，剩下的下次再讨论。

## 参考文献

[1] Qt帮助文档, 搜索关键词 Signals & Slots
[2] IBM文档库 https://www.ibm.com/developerworks/cn/linux/guitoolkit/qt/signal-slot/index.html


