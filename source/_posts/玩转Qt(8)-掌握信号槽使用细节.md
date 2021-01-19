---
title: 玩转Qt(8)-掌握信号槽使用细节
photos: /images/Qt4/signals-slot.png
tags:
  - Qt
  - 信号-槽
  - Qt实用技能
categories: 玩转Qt
abbrlink: 64079
date: 2019-09-02 12:44:23
---

- [简介](#%e7%ae%80%e4%bb%8b)
- [信号与槽的声明](#%e4%bf%a1%e5%8f%b7%e4%b8%8e%e6%a7%bd%e7%9a%84%e5%a3%b0%e6%98%8e)
- [信号-槽的使用](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e7%9a%84%e4%bd%bf%e7%94%a8)
  - [信号的使用](#%e4%bf%a1%e5%8f%b7%e7%9a%84%e4%bd%bf%e7%94%a8)
  - [槽函数的使用](#%e6%a7%bd%e5%87%bd%e6%95%b0%e7%9a%84%e4%bd%bf%e7%94%a8)
  - [信号-槽的"元调用"](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e7%9a%84%22%e5%85%83%e8%b0%83%e7%94%a8%22)
- [信号和信号的参数](#%e4%bf%a1%e5%8f%b7%e5%92%8c%e4%bf%a1%e5%8f%b7%e7%9a%84%e5%8f%82%e6%95%b0)
  - [注册元类型](#%e6%b3%a8%e5%86%8c%e5%85%83%e7%b1%bb%e5%9e%8b)
- [信号-槽的连接 connect函数](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e7%9a%84%e8%bf%9e%e6%8e%a5-connect%e5%87%bd%e6%95%b0)
  - [连接的不同写法](#%e8%bf%9e%e6%8e%a5%e7%9a%84%e4%b8%8d%e5%90%8c%e5%86%99%e6%b3%95)
    - [元方法式](#%e5%85%83%e6%96%b9%e6%b3%95%e5%bc%8f)
    - [函数指针式](#%e5%87%bd%e6%95%b0%e6%8c%87%e9%92%88%e5%bc%8f)
      - [函数重载的处理](#%e5%87%bd%e6%95%b0%e9%87%8d%e8%bd%bd%e7%9a%84%e5%a4%84%e7%90%86)
    - [functor式](#functor%e5%bc%8f)
      - [关于functor](#%e5%85%b3%e4%ba%8efunctor)
      - [functor](#functor)
  - [connect的连接类型](#connect%e7%9a%84%e8%bf%9e%e6%8e%a5%e7%b1%bb%e5%9e%8b)
  - [connect的返回值](#connect%e7%9a%84%e8%bf%94%e5%9b%9e%e5%80%bc)


# 简介

之前的文章《认清信号槽的本质》、《窥探信号槽的实现细节》讨论了一些原理，

这次我们来讨论一些信号-槽的使用细节。

# 信号与槽的声明

要使用信号-槽功能，先决条件是继承QObject类，并在类声明中增加Q_OBJECT宏。

之后在"signals:" 字段之后声明一些函数，这些函数就是信号。

在"public slots:" 之后声明的函数，就是槽函数。

例如下面的代码:

```c++
//Tom.h
#pragma once

#include <QObject>
#include <QDebug>
class Tom : public QObject
{
    Q_OBJECT
public:
    Tom(QObject *parent = nullptr) : QObject(parent)
    {
    }
    void miaow()
    {
        qDebug() <<  u8"喵!" ;
        emit miao();
    }
signals:
    void miao();
};
```
```C++
//Jerry.h
#pragma once

#include <QObject>
#include <QDebug>
class Jerry : public QObject
{
    Q_OBJECT
public:
    Jerry(QObject *parent = nullptr) : QObject(parent)
    {
    }
public slots:
    void runAway()
    {
        qDebug() << u8"那只猫又来了，快溜！" ;
    }
};
```
# 信号-槽的使用

使用比较简单，先说一下使用。

## 信号的使用

信号 就是普通的类成员函数，信号只要声明(declare)，不需要实现(implement)，实现由moc(元对象编译器)自动生成。

信号的触发，可以用emit，也可以直接调用函数。

例如:
```C++
	//实例化一个tom对象
	Tom tom;
	//通过emit发射信号
	emit tom.miao();
	//直接调用信号。效果和emit一样。
	tom.miao();
```

Qt源码的qobejctdefs.h头文件中，可以看到emit宏其实是空的。

```C++
//qobejctdefs.h
#ifndef QT_NO_EMIT
# define emit
#endif
```

## 槽函数的使用

槽函数和普通的成员函数一样。。。

## 信号-槽的"元调用"

信号-槽特殊的地方，是moc(元对象编译器)为其生成了一份"元信息",可以通过QMetaObject::invokeMethod的方式调用

例如:
```C++
	//实例化一个tom对象
	Tom tom;
	//通过invok方式调用信号
	QMetaObject::invokeMethod(&tom, "miao");
	
	//实例化一个jerry对象
	Jerry jerry;
	//通过invok方式调用槽
	QMetaObject::invokeMethod(&jerry, "runAway");
```

一般在知道如何声明qobject的场景，没必要多此一举用invoke。

在一些需要"运行期反射"的情况下(头文件都没有,只知道有这么个对象,和函数的名字)，invoke十分有用。

invokeMethod还可以带参数、可以获取返回值，这不是本文的重点，这里就不展开了，详细的可以参考Qt帮助文档和元对象系统。

# 信号和信号的参数

信号可以带参数，参数的类型，必须是元对象系统能够识别的类型, 即元类型。（元对象系统后面再细说）

## 注册元类型

Qt已经将大部分常用的基础类型，都注册进了元对象系统，可以在QMetaType类中看到。

通常写的继承于QObject的子类，本身已经附带了元信息，可以直接在信号-槽中使用。

不是继承于QObject的结构体、类等自定义类型，可以通过Q_DECLARE_METATYPE宏 和 qRegisterMetaType函数进行注册，之后就可以在信号-槽中使用。

例如：
```C++
  struct MyStruct
  {
      int i;
      ...
  };

  Q_DECLARE_METATYPE(MyStruct)
```
或者带命名空间的：
```C++
  namespace MyNamespace
  {
      ...
  }

  Q_DECLARE_METATYPE(MyNamespace::MyStruct)
```

这里说明一下细节，Q_DECLARE_METATYPE宏声明过后，只是生成了元信息，可以被QVariant识别，还不能

用于队列方式的信号、槽，需要用qRegisterMetaType进行注册。而qRegisterMetaType要求"全定义"，也就是

提供类的"复制构造函数"和"赋值操作符"。

前面那种简单类型，C++编译器默认提供浅拷贝的"复制构造函数"和"赋值操作符"实现，可以直接用。

```c++
  struct MyStruct
  {
      int i;
  };
```

而复杂一些的类，就要提供"全定义"。

(顺带一提，信号的参数可以是任意注册过的对象，而C++11的lambda、std::bind也是对象，只要注册过，也是可以通过信号参数发送出去的。)

# 信号-槽的连接 connect函数

信号与槽，通过connect函数进行连接，之后就可以用信号去触发槽函数了。

连接的一般格式是Connectin = connect(obj1, signal1, obj2, slot1, connectType);

## 连接的不同写法

connect函数重载实现了多种不同的参数写法，以Qt5.12为例，大致分为三类:
 
 元方法式、函数指针式、functor式
 
### 元方法式

元方法式是最常用的写法，函数声明如下：

```C++
    //connect(1) 字符串式信号槽
    static QMetaObject::Connection connect(const QObject *sender, const char *signal,
                        const QObject *receiver, const char *member, Qt::ConnectionType = Qt::AutoConnection);
    //connect(2) QMetaMethod式信号槽
    static QMetaObject::Connection connect(const QObject *sender, const QMetaMethod &signal,
                        const QObject *receiver, const QMetaMethod &method,
                        Qt::ConnectionType type = Qt::AutoConnection);
	//connect(3) 对(1)的重载, 非static去掉receiver
	inline QMetaObject::Connection connect(const QObject *sender, const char *signal,
                        const char *member, Qt::ConnectionType type = Qt::AutoConnection) const;
```

Qt应用程序中用到最多的是connect(1)的写法，例如:

```C++
	Tom tom;
	Jerry jerry;
	connect(&tom, SIGNAL(miao()), &jerry, SLOT(runAway()))
```
其中SIGNAL、SLOT两个宏, 作用是将函数转换成字符串。

connect(1)的实现是靠字符串去查找元方法，以实现连接。

connect(2) 则是把信号槽的字符串换成了元方法QMetaMethod, 一般不会直接用这种写法。

connect(3)是对connect(1)的重载，非静态成员函数，本身有this指针，所以省略了receiver参数。

### 函数指针式

函数指针式写法，声明如下：

```C++
	//connect(4) 连接信号到qobject的成员函数
	template <typename Func1, typename Func2>
    static inline QMetaObject::Connection connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal,
                                     const typename QtPrivate::FunctionPointer<Func2>::Object *receiver, Func2 slot,
                                     Qt::ConnectionType type = Qt::AutoConnection);
									 
	//connect(5) 连接信号到非成员函数。
	template <typename Func1, typename Func2>
    static inline typename std::enable_if<int(QtPrivate::FunctionPointer<Func2>::ArgumentCount) >= 0, QMetaObject::Connection>::type
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, Func2 slot);
    
	//connect(6) 连接信号到非成员函数。比(5)多一个context,可以设置连接类型
	template <typename Func1, typename Func2>
    static inline typename std::enable_if<int(QtPrivate::FunctionPointer<Func2>::ArgumentCount) >= 0 &&
                                          !QtPrivate::FunctionPointer<Func2>::IsPointerToMemberFunction, QMetaObject::Connection>::type
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, const QObject *context, Func2 slot,
                    Qt::ConnectionType type = Qt::AutoConnection);			
```
connect(4)用的也比较多，用法如下:
```C++
	Tom tom;
	Jerry jerry;
	connect(&tom, &Tom::miao, &jerry, &Jerry::runAway);
```
信号-槽换成了C++的 取成员函数指针 的形式。

connect(4)本身的实现，比connect(1)快一些，因为省去了字符串查找的过程。

而连接建立后，从信号触发到槽函数的执行，两种写法是没有区别的。

在一些需要"运行期反射"的情况下(头文件都没有,只知道有这么个对象,和函数的名字),只能用connect(1)。

connect(5)可以连接信号到任意非成员函数指针上。除了槽函数，普通的函数也可以连接。这种连接不支持设置连接类型，可以看作是单纯的函数调用。

connect(6)是对connect(5)的重载,增加了一个context对象代替reveicer对象的作用。这种连接是可以设置连接类型的。

#### 函数重载的处理

信号-槽函数有重载的情况下，写函数指针式connect会报错，就需要类型转换。

比如：QLocalSocket有一个成员函数error,也有一个信号error,直接写connect会报错的。

Qt为我们提供了QOverload这个模板类，以解决这个问题。

```C++
//连接重载过的函数，使用QOverload做leixing 转换
connect(socket, QOverload<QLocalSocket::LocalSocketError>::of(&QLocalSocket::error), this, &XXX::onError);
```
编译器支持C++14，还可以用qOverload模板函数
```C++
//连接重载过的函数，使用QOverload做leixing 转换
connect(socket, qOverload<QLocalSocket::LocalSocketError>(&QLocalSocket::error), this, &XXX::onError);
```

还有像QNetworkReply::error、QProcess::finished等等，都有重载，用的时候要转换处理一下。

### functor式

#### 关于functor
问: 什么是functor？functor有什么用?

答: 在C++11之前, Qt通过自己的实现来推导函数指针及其参数，即QtPrivate::FunctionPointer, 用来处理信号-槽的连接。

C++11带来了lambda, 以及std::bind和std::function, std::function本身可以存储lambda、std::bind以及FunctionPointer。

这时候Qt已有的connect(4)、connect(5)、connect(6)是可以支持FunctionPointer的,而新出现的lambda以及std::bind是不支持的，

QtPrivate::FunctionPointer推导不出这些类型。所以Qt把这些不支持的新类型(主要是lambda和std::bind)称为functor(文档和源码都这么命名)，

并增加了connect(7)和connect(8)以支持functor。

#### functor

functor式写法，声明如下：

```C++
	//connect(7) 连接信号到任意functor
    template <typename Func1, typename Func2>
    static inline typename std::enable_if<QtPrivate::FunctionPointer<Func2>::ArgumentCount == -1, QMetaObject::Connection>::type
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, Func2 slot);
			
    //connect(8) 连接信号到任意functor。比(7)多一个context,可以设置连接类型
	template <typename Func1, typename Func2>
    static inline typename std::enable_if<QtPrivate::FunctionPointer<Func2>::ArgumentCount == -1, QMetaObject::Connection>::type
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, const QObject *context, Func2 slot,
                    Qt::ConnectionType type = Qt::AutoConnection);			
```

connect(7)可以连接信号到任意lambda、std::bind上。

connect(8)是对(7)的重载，增加了一个context对象代替reveicer对象的作用。这种连接是可以设置连接类型的。

## connect的连接类型

connectType为连接类型，默认为AutoConnection，即Qt自动处理，大部分情况下也不用管。个别情况，需要手动指定。

可选的连接类型有
自动 AutoConnection
直连 DirectConnection 
队列 QueuedConnection
唯一连接 UniqueConnection

自动处理的逻辑是，如果发送信号的线程和receiver在同一个线程，就是DirectConnection(直接函数调用),不是同一个线程，则转换为QueuedConnection。

```doc

//引用自《Qt原理-窥探信号槽的实现细节》

如果信号-槽连接方式为QueuedConnection，不论是否在同一个线程，按队列处理。

如果信号-槽连接方式为Auto，且不在同一个线程，也按队列处理。

如果信号-槽连接方式为阻塞队列BlockingQueuedConnection，按阻塞处理。
 
(注意同一个线程就不要按阻塞队列调用了。因为同一个线程，同时只能做一件事，

本身就是阻塞的，直接调用就好了，如果走阻塞队列，则多了加锁的过程。如果槽中又发了

同样的信号，就会出现死锁：加锁之后还未解锁，又来申请加锁。)

队列处理，就是把槽函数的调用，转化成了QMetaCallEvent事件，通过QCoreApplication::postEvent

放进了事件循环, 等到下一次事件分发，相应的线程才会去调用槽函数。
```

下面举例一些需要手动指定连接类型的场景：

例1-跨多个线程：

A线程中写connect，让B线程中的信号连到C线程的槽中，希望C的槽在C中执行。

这种情况要明确指定QueuedConnection，不写的话按照Auto处理，C中的槽会在A中执行。

例2-跨线程DirectConnection

(这种用法在Qml的渲染引擎SceneGraph中比较常见)。

A线程为内部代码，不能修改，一些特定的节点会有信号发出。

B线程为用户代码，有一些功能函数，希望在A线程中去执行。

这种情况，将A的信号连接到B的函数，连接方式指定为DirectConnection，就可以把B的函数插入到A线程发信号的地方了。

效果类似于子类重载父类的函数。

## connect的返回值

connect的返回值为QMetaObject::Connection,代表一个连接。大部分情况下，不用管返回值。

Connection可以用来验证链接是否有效，可以用来断开连接。

一般用disconnect函数就可以断开连接；而signal-functor的这种形式的连接，没有object的存在，只能用Connection断开。