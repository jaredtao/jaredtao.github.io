---
title: 玩转Qml(5)-Qml与C++交互
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: 玩转Qml
abbrlink: 64048
date: 2019-05-17 20:37:33
---

- [简介](#简介)
- [源码](#源码)
- [C++访问Qml](#c访问qml)
  - [findChild](#findchild)
  - [QQmlComponent](#qqmlcomponent)
- [Qml访问C++](#qml访问c)
  - [注册类并使用](#注册类并使用)
  - [注册实例并使用](#注册实例并使用)

## 简介

本文是《玩转Qml》系列文章的第五篇，涛哥将教大家，Qml与C++的交互。

Qml已经有很多功能，不过终归会有不够用或不适用的地方，需要通过与C++的交互进行功能扩展。

这回涛哥尝试把所有Qml与C++交互相关的知识点都写出来，做一个透彻、全面的总结。

## 源码

《玩转Qml》系列文章，配套了一个优秀的开源项目:TaoQuick

github https://github.com/jaredtao/TaoQuick

访问不了或者速度太慢，可以用国内的镜像网站gitee

https://gitee.com/jaredtao/TaoQuick


## C++访问Qml

c++访问Qml有两种方式： findChild和 QQmlComponent。

### findChild

了解Qt的人都知道，Qt的很多对象是QObject的子类，这些QObject只要设置了parent，就是有父子关系的，会产生一棵 "对象树"。

只要有了根节点，树上的任意节点都可以通过findChild的方式获取到。

写个简单的TaoObject，来示意一下:

```C++
class TaoObject
{
public:
    //构造函数，传递parent进来
    TaoObject(TaoObject *parent = nullptr) : m_pParent(parent)
    {
        if (m_pParent)
        {
            m_pParent->appendChild(this);
        }
    }
    //析构函数，析构children。即子对象自动回收机制。
    ~TaoObject()
    {
        for (auto *pObj : m_children)
        {
            delete pObj;
        }
        m_children.clear();
    }
    //获取name
    const QString &getName() const
    {
        return m_name;
    }
    //设置name
    void setName(const QString &name)
    {
        m_name = name;
    }
    //查找子Object
    TaoObject *findChild(const QString &name)
    {
        //先检查自己的名字，是否匹配目标名字
        if (m_name == name)
        {
            return this;
        }
        //遍历子Object，查找
        for (auto pObj : m_children)
        {
            //递归调用，深度优先的搜索
            auto resObj = pObj->findChild(name);
            if (resObj)
            {
                return resObj;
            }
        }
            return nullptr;
    }
protected:
    void appendChild(TaoObject *child)
    {
        m_children.push_back(child);
    }

private:
    //存储名字
    QString m_name;
    //子对象列表
    std::vector<TaoObject *> m_children;
    //父对象指针
    TaoObject *m_pParent = nullptr;
};

```

Qml的基本元素,大多是继承于QQuickItem，而QQuickItem继承于QObject。

所以Qml大多数对象都是QObject的子类，也是可以通过findChild的方式获取到对象指针。

拿到了QObject，可以通过qobject_cast转换成具体的类型来使用，也可以直接用QObject的invok方法。

例如有如下Qml代码:
```Qml
Item {
    id: root
    ...
    Rectangle {
        id: centerRect
        objectName: "centerRect"        //必不可少的objectName
        property bool canSee: visible   //自定义属性，可以被C++ invok访问
        signal sayHello()               //自定义信号1，可以被C++ invok调用
        signal sayHelloTo(name)         //自定义信号2，带参数。可以被C++ invok调用。参数的名字要起好，后面通过这个名字来使用参数
        function rotateToAngle(angle)   //自定义js函数，旋转至指定角度。可以被C++ invok调用。
        {
            rotation = angle
            return true;
        }
    }
    ...
}

```
那么在C++ 中访问的方式是：

* 如果用QQuickView加载qml，就是

```C++
QQuickView view;
...
QObject *centerObj = view.rootObject()->findChild<QObject *>("centerRect");
if (!centerObj) { return;}
```
* 如果用QQmlEngine加载qml，就是
```c++
QQmlEngine engine;
...
QObject *centerObj = engine.rootObject()->findChild<QObject *>("centerRect");
if (!centerObj) { return;}
```
(QObject类型也可以换成QQuickItem 或者其它)

拿到了对象指针，接下来就好办了

访问其属性

```C++
bool canSee = centerObj->property("canSee").toBool();
```

发射其信号(其实就是函数调用)

```C++
QObject::invokeMethod(centerObj, "sayHello");
QObject::invokeMethod(centerObj, "sayHelloTo", Q_ARG(QString, "Tao"))
```

调用其js函数,可以传参数过去,可以取得返回值

```C++
bool ok;
QObject::invokeMethod(centerObj, "rotateToAngle", Q_RETURN_ARG(bool, ok), Q_ARG(qreal 180));
```

这里再补充一下, Qml中给自定义的信号写槽或连接到别的槽(Qml中的槽就是js函数):

```qml
Item {
    id: root
    ...
    Rectangle {
        id: centerRect
        objectName: "centerRect" //必不可少的objectName
        signal sayHello()       //自定义信号1
        signal sayHelloTo(name) //自定义信号2，带参数。参数的名字要起好，后面通过这个名字来使用参数

        function rotateToAngle(angle) //自定义js函数，旋转至指定角度
        {
            rotation = angle
        }

        onSayHello: {               //信号1的槽
            console.log("hello")
        }
        onSayHelloTo: {             //信号2的槽，直接用信号定义时的参数名字name作为关键字访问参数
            console.log("hello", name)
        }
        Component.onCompleted: { 
            //信号2 连接到 root的函数。参数会自动匹配。
            sayHello.connect(root.rootSayHello)
        }
    }
    ...

    function rootSayHello(name) {
        console.log("root: hello", name)
    }
}
```
### QQmlComponent

C++中的QQmlComponent可以用来动态加载Qml文件，并可以创建多个实例，

对应Qml中的Component。Qml中还有一个Loader，也可以动态加载并创建单个实例。

(QQmlComponent这种方式不太多见，不过涛哥之前参与过开发一个框架，使用的就是QQmlComponent动态加载Qml，

完全在c++中控制界面的加载，加载效率、内存占用上都比纯Qml优秀。)

来看一个例子：
```qml
// Circle.qml
Rectangle {
    width: 300
    height: width
    radius: width / 2
    color: "red"
}
```

```C++
QQmlEngine engine;
QQmlComponent component(&engine, QUrl::fromLocalFile("Circle.qml"));

QObject *circleObject = component.create();
QQuickItem *item = qobject_cast<QQuickItem*>(circleObject);
int width = item->width();
```
拿到对象指针，就和前面的一样了，这里不再赘述了。

## Qml访问C++

Qml要访问C++的内容，需要先从C++把要访问的内容注册进Qml。

先说说能用哪些：

注册过后，Qml中可以访问的内容，包括 Q_INVOKABLE 修饰的函数、枚举、 QObject的属性 信号 槽

Q_INVOKABLE 函数可以用在普通的结构体或者类中，但是这种用法不常见/不方便。常见的是在QObject的子类中，给非槽函数设置为Q_INVOKABLE

枚举的注册Qt帮助文档很详细，而且5.10以后可以在qml中定义枚举了，这里涛哥就不展开了。

QObject的属性 信号 槽，都是可以通过注册后，在qml中使用的。信号、槽都可以带参数，槽可以有返回值。

```C++
class BrotherTao : public QObject 
{
    Q_OBJECT //这个宏一定要写上。不写可能的后果是，moc生成失败，信号 槽实现不了，编译过不了。
    Q_PROPERTY(QString name READ name WRITE setName NOTIFY nameChanged) //自定义属性，操作包括： 读、写和通知。Qml可以读写、获取通知
public:
    ...
    //唱歌
    Q_INVOKABLE void sing();        //invok函数，可以被Qml调用。可以带参数和返回值
public slots:
    //打游戏，参数为次数，返回值为得分。
    int playGame(int count);        //槽函数，可以被Qml调用。可以带参数和返回值

signals:
    //肚子饿了。参数为想吃的东西。
    void hungry(const QString &foodName);                  //信号，可以被Qml接收。
    ...
};
```
这里要说的是，属性、函数参数、返回值的类型，都需要是Qml能识别的类型。

Qt的常用类型已经在Qt内部注册好了，自定义的需要单独注册。

再说说怎么用：

注册分为两种：注册类型和注册实例。

### 注册类并使用

```C++
qmlRegisterTyle<BrotherTao>("BrotherTao",1, 0, "BrotherTao");
```
```qml
import BrotherTao 1.0
Item {
    BrotherTao {         //实例化一个对象
        id: tao
        onHungry: {     //给信号写个槽函数
            if (foodName === "蛋炒饭") {      //示意一下，不要在意吃啥。
                console.log("涛哥要吃蛋炒饭")
            } else if (foodName === "水饺") {
                console.log("涛哥要吃水饺")
            } 
        }
    }
    ...
    Button {
        onClicked: {
            //这个按钮按下的时候，涛哥开始唱歌
            tao.sing();
        }
    }
    Button {
        onClicked: {
            //这个按钮按下的时候，涛哥开始打游戏
            let score = tao.sing(3);
            console.log("涛哥打游戏次数"， 3， "得分为", score)
        }
    }
}
```
### 注册实例并使用

```c++
BrotherTao tao;     //C++中创建的实例

//如果用QQ'u'ic'kView加载Qml
QQuickView view;
...
view.rootContext()->setContextProperty("tao", &tao);    //注意这个名字不要用大写字母开头，规则和Qml中的id不能用大写字母开头一样。

//如果用QQmlEngine加载Qml
QQmlEngine engine;
...
engine..rootContext()->setContextProperty("tao", &tao); //注意这个名字不要用大写字母开头，规则和Qml中的id不能用大写字母开头一样。

```

```qml
//这种不用再import了
Item {
    Connections {        //通过connectins连接信号
        target: tao     //指定target
        onHungry: {     //给信号写个槽函数
            if (foodName === "蛋炒饭") {      //示意一下，不要在意吃啥。
                console.log("涛哥要吃蛋炒饭")
            } else if (foodName === "水饺") {
                console.log("涛哥要吃水饺")
            } 
        }
    }
    ...
    Button {
        onClicked: {
            //这个按钮按下的时候，涛哥开始唱歌
            tao.sing();
        }
    }
    Button {
        onClicked: {
            //这个按钮按下的时候，涛哥开始打游戏
            let score = tao.sing(3);
            console.log("涛哥打游戏次数"， 3， "得分为", score)
        }
    }
}
```



