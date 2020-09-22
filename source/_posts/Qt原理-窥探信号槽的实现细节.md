---
title: Qt原理-窥探信号槽的实现细节
photos: /img/avatar.jpg
tags:
  - Qt
  - 信号-槽
  - Qt原理
categories: Qt进阶之路
abbrlink: 50256
date: 2019-08-30 18:44:23
---

- [简介](#%e7%ae%80%e4%bb%8b)
- [猫和老鼠的故事](#%e7%8c%ab%e5%92%8c%e8%80%81%e9%bc%a0%e7%9a%84%e6%95%85%e4%ba%8b)
- [声明与实现](#%e5%a3%b0%e6%98%8e%e4%b8%8e%e5%ae%9e%e7%8e%b0)
- [Q_OBJECT宏](#qobject%e5%ae%8f)
- [信号的moc生成](#%e4%bf%a1%e5%8f%b7%e7%9a%84moc%e7%94%9f%e6%88%90)
- [信号的触发](#%e4%bf%a1%e5%8f%b7%e7%9a%84%e8%a7%a6%e5%8f%91)
- [槽和moc生成](#%e6%a7%bd%e5%92%8cmoc%e7%94%9f%e6%88%90)
- [第三方信号槽实现](#%e7%ac%ac%e4%b8%89%e6%96%b9%e4%bf%a1%e5%8f%b7%e6%a7%bd%e5%ae%9e%e7%8e%b0)


# 简介

本文是《Qt进阶之路》系列文章的特别篇，涛哥在这里讨论Qt信号-槽的实现细节。

上次的文章《Qt实用技能4-认清信号槽的本质》中介绍过，信号-槽是一种对象之间的通信机制，是

Qt在标准C++之外，使用元对象编译器(MOC)实现的语法糖。

这次通过一个简单的案例，学习一些信号-槽的实现细节。

# 猫和老鼠的故事

![预览](/images/Qt4/1.jpg)

还是拿上次的设定来说明：Tom有个技能叫"喵"，就是发出猫叫，而正在偷吃东西的Jerry,听见猫叫声就会逃跑。

我们用信号-槽的方式写出来。

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

以上面的代码为例，要使用信号-槽功能，先决条件是继承QObject类，并在类声明中增加Q_OBJECT宏。

之后在"signals:" 字段之后声明一些函数，这些函数就是信号。

在"public slots:" 之后声明的函数，就是槽函数。

接下来看看我们的main函数:

```C++
//main.cpp
#include <QCoreApplication>
#include "Tom.h"
#include "Jerry.h"
int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);

    Tom tom;
    Jerry jerry;

    QObject::connect(&tom, &Tom::miao, &jerry, &Jerry::runAway);
    tom.miaow();


    return a.exec();
}

```

信号-槽都准备好了，接下来创建两个对象实例，并使用QObject::connect将信号和槽连接起来。

最后使用emit发送信号，就会自动触发槽函数了。

运行结果：

![预览](/images/Qt5/1.jpg)

# 声明与实现

信号和槽的本质都是函数。

我们知道C++中的函数要有声明(declare)，也要有实现(implement),

而信号只要声明，不需要写实现。这是因为moc会为我们自动生成。

另外触发信号时，不写emit关键字，直接调用信号函数，也是没有问题的。

# Q_OBJECT宏

我们来看一下Q_OBJECT宏，展开如下：

(不同的Qt版本有些差异，涛哥这里用的是5.12.4，以此为例)

```C++
public: \
    QT_WARNING_PUSH \
    Q_OBJECT_NO_OVERRIDE_WARNING \
    static const QMetaObject staticMetaObject; \
    virtual const QMetaObject *metaObject() const; \
    virtual void *qt_metacast(const char *); \
    virtual int qt_metacall(QMetaObject::Call, int, void **); \
    QT_TR_FUNCTIONS \
private: \
    Q_OBJECT_NO_ATTRIBUTES_WARNING \
    Q_DECL_HIDDEN_STATIC_METACALL static void qt_static_metacall(QObject *, QMetaObject::Call, int, void **); \
    QT_WARNING_POP \
    struct QPrivateSignal {}; \
    QT_ANNOTATE_CLASS(qt_qobject, "")
```

我们看到，关键的地方，是声明了一个只读的静态成员变量staticMetaObject，以及3个public的成员函数
```C++
    static const QMetaObject staticMetaObject; 

    virtual const QMetaObject *metaObject() const; 

    virtual void *qt_metacast(const char *); 

    virtual int qt_metacall(QMetaObject::Call, int, void **);
```

还有一个private的静态成员函数qt_static_metacall

```c++
static void qt_static_metacall(QObject *, QMetaObject::Call, int, void **)
```

那么声明的这些成员变量/函数，在哪里实现？答案是moc生成的cpp文件。

# 信号的moc生成

![预览](/images/Qt5/2.jpg)

如上图所示目录结构，项目编译完成后，在build文件夹中，自动生成了moc_Jerry.cpp 和 moc_Tom.cpp两个文件

其中moc_Tom.cpp内容如下：

```C++
/****************************************************************************
** Meta object code from reading C++ file 'Tom.h'
**
** Created by: The Qt Meta Object Compiler version 67 (Qt 5.12.4)
**
** WARNING! All changes made in this file will be lost!
*****************************************************************************/

#include "../../TomJerry/Tom.h"
#include <QtCore/qbytearray.h>
#include <QtCore/qmetatype.h>
#if !defined(Q_MOC_OUTPUT_REVISION)
#error "The header file 'Tom.h' doesn't include <QObject>."
#elif Q_MOC_OUTPUT_REVISION != 67
#error "This file was generated using the moc from 5.12.4. It"
#error "cannot be used with the include files from this version of Qt."
#error "(The moc has changed too much.)"
#endif

QT_BEGIN_MOC_NAMESPACE
QT_WARNING_PUSH
QT_WARNING_DISABLE_DEPRECATED
struct qt_meta_stringdata_Tom_t {
    QByteArrayData data[3];
    char stringdata0[10];
};
#define QT_MOC_LITERAL(idx, ofs, len) \
    Q_STATIC_BYTE_ARRAY_DATA_HEADER_INITIALIZER_WITH_OFFSET(len, \
    qptrdiff(offsetof(qt_meta_stringdata_Tom_t, stringdata0) + ofs \
        - idx * sizeof(QByteArrayData)) \
    )
static const qt_meta_stringdata_Tom_t qt_meta_stringdata_Tom = {
    {
QT_MOC_LITERAL(0, 0, 3), // "Tom"
QT_MOC_LITERAL(1, 4, 4), // "miao"
QT_MOC_LITERAL(2, 9, 0) // ""

    },
    "Tom\0miao\0"
};
#undef QT_MOC_LITERAL

static const uint qt_meta_data_Tom[] = {

 // content:
       8,       // revision
       0,       // classname
       0,    0, // classinfo
       1,   14, // methods
       0,    0, // properties
       0,    0, // enums/sets
       0,    0, // constructors
       0,       // flags
       1,       // signalCount

 // signals: name, argc, parameters, tag, flags
       1,    0,   19,    2, 0x06 /* Public */,

 // signals: parameters
    QMetaType::Void,

       0        // eod
};

void Tom::qt_static_metacall(QObject *_o, QMetaObject::Call _c, int _id, void **_a)
{
    if (_c == QMetaObject::InvokeMetaMethod) {
        auto *_t = static_cast<Tom *>(_o);
        Q_UNUSED(_t)
        switch (_id) {
        case 0: _t->miao(); break;
        default: ;
        }
    } else if (_c == QMetaObject::IndexOfMethod) {
        int *result = reinterpret_cast<int *>(_a[0]);
        {
            using _t = void (Tom::*)();
            if (*reinterpret_cast<_t *>(_a[1]) == static_cast<_t>(&Tom::miao)) {
                *result = 0;
                return;
            }
        }
    }
    Q_UNUSED(_a);
}

QT_INIT_METAOBJECT const QMetaObject Tom::staticMetaObject = { {
    &QObject::staticMetaObject,
    qt_meta_stringdata_Tom.data,
    qt_meta_data_Tom,
    qt_static_metacall,
    nullptr,
    nullptr
} };


const QMetaObject *Tom::metaObject() const
{
    return QObject::d_ptr->metaObject ? QObject::d_ptr->dynamicMetaObject() : &staticMetaObject;
}

void *Tom::qt_metacast(const char *_clname)
{
    if (!_clname) return nullptr;
    if (!strcmp(_clname, qt_meta_stringdata_Tom.stringdata0))
        return static_cast<void*>(this);
    return QObject::qt_metacast(_clname);
}

int Tom::qt_metacall(QMetaObject::Call _c, int _id, void **_a)
{
    _id = QObject::qt_metacall(_c, _id, _a);
    if (_id < 0)
        return _id;
    if (_c == QMetaObject::InvokeMetaMethod) {
        if (_id < 1)
            qt_static_metacall(this, _c, _id, _a);
        _id -= 1;
    } else if (_c == QMetaObject::RegisterMethodArgumentMetaType) {
        if (_id < 1)
            *reinterpret_cast<int*>(_a[0]) = -1;
        _id -= 1;
    }
    return _id;
}

// SIGNAL 0
void Tom::miao()
{
    QMetaObject::activate(this, &staticMetaObject, 0, nullptr);
}
QT_WARNING_POP
QT_END_MOC_NAMESPACE

```

可以大致看出，生成的cpp文件中，就是变量staticMetaObject以及 那几个函数的实现。

staticMetaObject是一个结构体，用来存储Tom这个类的信号、槽等元信息，并把

qt_static_metacall静态函数作为函数指针存储起来。

因为是静态成员，所以实例化多少个Tom对象，它们的元信息都是一样的。

qt_static_metacall函数提供了两种“元调用的实现”：

如果是InvokeMetaMethod类型的调用，则直接 把参数中的QObject对象，

转换成Tom类然后调用其miao函数

如果是IndexOfMethod类型的调用，即获取元函数的索引号，则计算miao函数的偏移并返回。


而moc_Tom.cpp末尾的

```C++
// SIGNAL 0
void Tom::miao()
{
    QMetaObject::activate(this, &staticMetaObject, 0, nullptr);
}
```
就是信号函数的实现。

# 信号的触发

miao信号的实现，直接调用了QMetaObject::activate函数。其中0代表miao这个函数的索引号。

QMetaObject::activate函数的实现，在Qt源码的QObject.cpp文件中，略微复杂一些，

且不同版本的Qt，实现差异都比较大，这里总结一下大致的实现：

先找出与当前信号连接的所有对象-槽函数，再逐个处理：

这里处理的方式，分为三种：

```C++
if((c->connectionType == Qt::AutoConnection && !receiverInSameThread)
                || (c->connectionType == Qt::QueuedConnection)) {
    // 队列处理
} else if (c->connectionType == Qt::BlockingQueuedConnection) {
    // 阻塞处理
    // 如果同线程，打印潜在死锁。
} else {
    //直接调用槽函数或回调函数
}
```
receiverInSameThread表示当前线程id和接收信号的对象的所在线程id是否相等。

如果信号-槽连接方式为QueuedConnection，不论是否在同一个线程，按队列处理。

如果信号-槽连接方式为Auto，且不在同一个线程，也按队列处理。

如果信号-槽连接方式为阻塞队列BlockingQueuedConnection，按阻塞处理。
 
(注意同一个线程就不要按阻塞队列调用了。因为同一个线程，同时只能做一件事，

本身就是阻塞的，直接调用就好了，如果走阻塞队列，则多了加锁的过程。如果槽中又发了

同样的信号，就会出现死锁：加锁之后还未解锁，又来申请加锁。)

队列处理，就是把槽函数的调用，转化成了QMetaCallEvent事件，通过QCoreApplication::postEvent

放进了事件循环, 等到下一次事件分发，相应的线程才会去调用槽函数。

关于事件循环，可以参考之前的文章《Qt实用技能3-理解事件循环》

# 槽和moc生成

slot函数我们自己实现了，moc不会做额外的处理，所以自动生成的moc_Jerry.cpp文件中，

只有Q_OBJECT宏的展开，和前面的moc_Tom.cpp是一致的，不赘述了。

# 第三方信号槽实现

信号-槽是非常优秀的通信机制，但Qt的moc实现方式，被一些人诟病，所以他们造了新的轮子,比如：

https://woboq.com/blog/verdigris-qt-without-moc.html

http://sigslot.sourceforge.net/

https://github.com/NoAvailableAlias/nano-signal-slot

https://github.com/pbhogan/Signals
