---
title: 玩转Qml(9)-Model和View
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: 玩转Qml
abbrlink: 17496
date: 2019-05-24 18:44:23
---

- [简介](#简介)
- [源码](#源码)
- [界面、数据和逻辑分离](#界面数据和逻辑分离)
- [Qt内置的Model-View](#qt内置的model-view)
- [整数做model](#整数做model)
  - [关于delegate](#关于delegate)
  - [View与Repeater的区别](#view与repeater的区别)
- [ListModel](#listmodel)
  - [静态ListModel](#静态listmodel)
  - [动态ListModel](#动态listmodel)
- [XmlListModel](#xmllistmodel)
- [ObjectModel](#objectmodel)
- [C++导出Model](#c导出model)
  - [QList<T>](#qlistt)
  - [QJsonArray](#qjsonarray)
  - [QQmlPropertyMap](#qqmlpropertymap)
- [ListView缺失的灵魂](#listview缺失的灵魂)
  - [搜索与排序](#搜索与排序)
  - [选中](#选中)
  - [拖拽](#拖拽)
  - [特效](#特效)

## 简介

本文是《玩转Qml》系列文章的第九篇，涛哥将教大家，Qml中Model和View的知识。

## 源码

《玩转Qml》系列文章，配套了一个优秀的开源项目:TaoQuick

github https://github.com/jaredtao/TaoQuick

访问不了或者速度太慢，可以用国内的镜像网站gitee

https://gitee.com/jaredtao/TaoQuick

## 界面、数据和逻辑分离

界面架构的理念发展的非常快，主要在Web技术的驱动下，就有这么多架构：

MVC、MVP、 MVVM、 Flux、Redux。

架构太多太复杂，只要抓住一些关键点就够了：界面、数据和逻辑要分别处理，最终要能够正确处理用户输入并显示结果。

先来看一下Qt中提供的架构：

![预览](/images/Qml9/1.png)

Model代表数据，View代表界面，这个Delegate嘛，就是用来定制View的显示方式和Controll的调用，也应该算进View里面去。

这样看来Qt是M-V架构 ? 其实Qt算是MVC架构，这个Controll一般是自己实现的，和Model放在一起的。

不过Qt有信号/槽机制，在QtQuick中以属性绑定的方式出现。信号/槽相当于Gof设计模式中的[观察者模式](https://github.com/jaredtao/DesignPattern/tree/master/code/Behavior/Observer)，也相当于Flux中的订阅/发布模式。

涛哥按自己的实践和理解，画了一个Qt的Model-View架构草图：

![预览](/images/Qml9/2.png)

## Qt内置的Model-View

View包括 ListView、TableView、TreeView这三种

(ComboBox也可以算作ListView)

![预览](/images/Qml9/4.png)

对应的Model包括 ListModel、TableModel、TreeModel

![预览](/images/Qml9/3.png)

Qt提供了一些抽象的Model类，需要自己去继承并实现接口，也有一些可以直接用。

下图是涛哥整理的Qt中model继承关系：

![预览](/images/Qml9/5.png)

其中的QStringListModel不是抽象类，可以直接用在ListView中。

QStandardItemModel也不是抽象类，可以直接用在任意一种View中。

在数据量大、有性能要求的地方，需要继承QAbstractItemModel类，重新实现一个model。

对于性能要求不高的数据展示，会有一些更加方便、取巧的方式，接着往下看吧。

(友情提示：涛哥不关心QWidget，只说QtQuick/Qml)


## 整数做model

在ListView中，一个整数作为model，就可以创建多个delegate实例。

整数作为model，也可以用在GridView、Combobox、Repeater等需要model的地方。

<Qml组件化编程6-进度条定制>一文中，展示渐变效果，就用的整数作为model

![预览](/images/Qml6/Gradiant.gif)

```Qml
import QtQuick 2.9
import QtQuick.Controls 2.5

Item {
    anchors.fill: parent

    GridView {
        id: g
        anchors.fill: parent
        anchors.margins: 20
        cellWidth: 160
        cellHeight: 160
        model: 180          //这里的数据Model直接给个整数180
        clip: true
        property var invalidList: [27, 39, 40, 45, 71, 74, 105, 111, 119, 130, 135, 141]    //这几个是不能用的，看过运行报错后手动列出来的。
        delegate: Item{
            width: 160
            height: 160
            Rectangle{
                width: 150
                height: 150
                anchors.centerIn: parent
                color: "white"
                radius: 10
                Text {
                    anchors.horizontalCenter: parent.horizontalCenter
                    anchors.top: parent.top
                    anchors.topMargin: 2
                    text: index + 1
                }
                Rectangle {
                    width: 100
                    height: width
                    radius:  width / 2
                    //编号在列表里的，直接渐变赋值为null，就不会在Qml运行时报警告了
                    gradient: g.invalidList.indexOf(modelData + 1) < 0 ? modelData + 1 : null
                    anchors.centerIn: parent
                    anchors.verticalCenterOffset: 10
                }
            }
        }
    }
}
```
### 关于delegate

简单说一下delegate：

上面GridView的 model设置为180，表示这个View要产生180个相同的构件实例，按照Grid的方式布局排列。

而delegate就相当于是一个模板，用来描述这180个相同的构件长啥样。当然每个实例不可能完全长得一样，我们可以通过

绑定delegate提供的内置属性或其它属性，达到"大同小异"的目的。

delegate中一般会提供一个index和一个modelData，详细的说明需要参考相应的View文档。

### View与Repeater的区别

上面的GridView虽然会创建180个实例，但并不是一次创建全部的，而是只创建能看见的那几个，否则会占用很多CPU、内存和GPU资源。

而Repeater这种就是直接生成180个，并没有做任何内置处理。

(Repeater也可以通过自己控制visible的方式，实现部分创建，后面涛哥有个RingView特效会用这种方式)

## ListModel

Qml提供了ListModel这样的一个封装，可以直接在Qml中定义静态的model

### 静态ListModel
```Qml
  import QtQuick 2.0

  ListModel {
      id: fruitModel

      ListElement {
          name: "Apple"
          cost: 2.45
      }
      ListElement {
          name: "Orange"
          cost: 3.25
      }
  }
```
然后在ListView中使用
```Qml
ListView {
    anchors.fill: parent
    model: fruitModel
    delegate: Item {
        Text {
            text: modelData.name
        }
        Text {
            text: cost
        }
    }
}
```
第一个text通过modelData.name获取到name值

第二个text直接用了cost，其实是modelData.cost省略了modelData。这种写法在静态的ListModel中是可以用的。

### 动态ListModel

ListModel还提供了一些动态修改数据的接口:

![预览](/images/Qml9/6.png)

像append、 set、insert这些，参数里的jsobject就是js环境中的Object类型，可以参考[JS手册](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object)

这里涛哥示例一下，动态添加元素

```Qml
...
onClicked: {
    var banana = new Object()
    
    //或者这样也行，按照js语法即可
    //var bababa = Object.create(null)

    banana["name"]="banana" //方括号 + key的方式设置成员
    babana.cost=15  //点+名字的方式设置成员
    
    fruitModel.append(banana)   //将创建的banana添加到model
}
...
```
更详细的用法，可以参考 涛哥两年前写过的一个[Qml表格编辑器](https://github.com/jaredtao/TableEdit)

里面有ListModel的JSON序列化和反序列化、动态增、删、改，Ubuntu风格的查找、Redo、UnDo等大部分功能。

TaoQuick项目的插件机制，也是通过JSON动态添加Model元素。[TaoQuick](https://github.com/jaredtao/TaoQuick)

## XmlListModel

处理xml的model，可以方便地使用XPath。

```Qml
  XmlListModel {
       id: feedModel
       source: "http://rss.news.yahoo.com/rss/oceania"
       query: "/rss/channel/item"
       XmlRole { name: "title"; query: "title/string()" }
       XmlRole { name: "link"; query: "link/string()" }
       XmlRole { name: "description"; query: "description/string()" }
  }
```

## ObjectModel

可视对象的集合，做为model，连Delegate都省了

```
  import QtQuick 2.0
  import QtQml.Models 2.1

  Rectangle {
      ObjectModel {
          id: itemModel
          Rectangle { height: 30; width: 80; color: "red" }
          Rectangle { height: 30; width: 80; color: "green" }
          Rectangle { height: 30; width: 80; color: "blue" }
      }

      ListView {
          anchors.fill: parent
          model: itemModel
      }
  }
```
## C++导出Model

除了以上这些，C++中导出的一些类型也可以作为数据model。

这里的导出包括Q_PROPERTY和 Q_INVOKABLE函数的返回值、槽函数的返回值，以及

setContextProperty注册到上下文的可用作model的类型。

一般使用Q_PROPERTY (本质上也是属性的get函数返回值，在js中做了转换)

### QList<T>

QList<QString> 字符串列表，可以直接用，不用多说了。

QList<QObject*> QObject列表，List中的任意一个QObject有一些属性变更时，都能通知到Qml。

### QJsonArray

QJsonArray也是可以直接导出给ListView用，不过注意是只读的。

### QQmlPropertyMap

QQmlPropertyMap 是一个Map结构, 但是这个结构注册后，Qml中可以直接用"点 + 名字"的方式访问其中的数据

```C++
  // create our data
  QQmlPropertyMap ownerData;
  ownerData.insert("name", QVariant(QString("John Smith")));
  ownerData.insert("phone", QVariant(QString("555-5555")));

  // expose it to the UI layer
  QQuickView view;
  QQmlContext *ctxt = view.rootContext();
  ctxt->setContextProperty("owner", &ownerData);

  view.setSource(QUrl::fromLocalFile("main.qml"));
  view.show();
```
```Qml
//main.qml
Text { 
    text: owner.name + " " + owner.phone 
}
```
## ListView缺失的灵魂

Qml这个ListView是残缺不全的，很多功能都要自己实现。

### 搜索与排序

前面提到的QSortFilterProxyModel是一种在数据上实现排序和过滤的方法。

还有一种在View层实现搜索和过滤的方式，即DelegateModelGroup。（已经有案例在用，后续再放出代码）

当然Qt5.12的ListView/TableView提供了行和列 隐藏控制的功能，View层做搜索会更方便一些。（还没有实践）

### 选中

按住Ctrl 再鼠标点击，多选， 再点击一下反选。

按住Shift再鼠标点击，连选。

旧的QtQuick.Controls 1中也有一个ListView，带SelectonModel功能，直接支持多选、反选。

5.12开始，QtQuick.Controls 1模块被废弃了，而Controls2中的ListView不带这功能了。只能自己记键盘按键来模拟实现。

(顺便吐槽一下，5.12直接把Controls 1的TreeView废掉了，Controls 2又没有TreeView。Controls 1的那个虽然还能用，程序跑起来就是一堆js 异常)

### 拖拽

拖动和放置功能也得自己做。

### 特效

ListView提供过度动画，下拉刷新一类的效果很多人已经做了，涛哥就不重复了。

(涛哥正在给TaoQuick开发高级插件TaoEffect，将会包含大量酷炫特效组件，敬请期待)

