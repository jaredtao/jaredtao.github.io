---
title: 玩转Qml(8)-Qml属性
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: 玩转Qml
abbrlink: 47000
date: 2019-05-22 19:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [Qml内置类型](#qml%E5%86%85%E7%BD%AE%E7%B1%BB%E5%9E%8B)
  - [简单类型](#%E7%AE%80%E5%8D%95%E7%B1%BB%E5%9E%8B)
  - [枚举](#%E6%9E%9A%E4%B8%BE)
  - [list](#list)
  - [var](#var)
    - [var数组](#var%E6%95%B0%E7%BB%84)
    - [var回调函数](#var%E5%9B%9E%E8%B0%83%E5%87%BD%E6%95%B0)
- [Qml模块扩展类型](#qml%E6%A8%A1%E5%9D%97%E6%89%A9%E5%B1%95%E7%B1%BB%E5%9E%8B)
- [Qml属性](#qml%E5%B1%9E%E6%80%A7)
  - [属性的change信号](#%E5%B1%9E%E6%80%A7%E7%9A%84change%E4%BF%A1%E5%8F%B7)
  - [属性绑定](#%E5%B1%9E%E6%80%A7%E7%BB%91%E5%AE%9A)
  - [动态解绑、动态绑定](#%E5%8A%A8%E6%80%81%E8%A7%A3%E7%BB%91%E5%8A%A8%E6%80%81%E7%BB%91%E5%AE%9A)
  - [条件绑定](#%E6%9D%A1%E4%BB%B6%E7%BB%91%E5%AE%9A)
  - [只读属性](#%E5%8F%AA%E8%AF%BB%E5%B1%9E%E6%80%A7)
  - [默认属性](#%E9%BB%98%E8%AE%A4%E5%B1%9E%E6%80%A7)
  - [属性别名](#%E5%B1%9E%E6%80%A7%E5%88%AB%E5%90%8D)
  - [QQmlProperty](#qqmlproperty)
- [下期预告](#%E4%B8%8B%E6%9C%9F%E9%A2%84%E5%91%8A)

## 简介

本文是《玩转Qml》系列文章的第八篇，涛哥将教大家，一些Qml中属性的知识和使用技巧。

## 源码

《玩转Qml》系列文章，配套了一个优秀的开源项目:TaoQuick

github https://github.com/jaredtao/TaoQuick

访问不了或者速度太慢，可以用国内的镜像网站gitee

https://gitee.com/jaredtao/TaoQuick


## Qml内置类型

说Qml属性之前，先来看看Qml中都有哪些类型吧

Qml本身支持的类型如下图：

![预览](/images/Qml8/1.png)

一共9个，可以在Qml中实用。

### 简单类型

```qml
Item {
    property bool doorIsOpened: true

    property int doorCount: 1 + 2 * 3

    property double PI: 3.1415926

    property real PI: 3.1415926

    property string name: "JaredTao"

    property url address: "https://jaredtao.github.io"
}

```
bool double int real string url 这6个简单的类型，C++中也分别有对应的类型，其中string对应QString，url对应QUrl，就不用多说了。

这里提一下，"1 + 2 * 3" 这种可以在编译期间确定的简单数值表达式，

Qml引擎会自动帮你计算成7。编译进二进制文件的时候就是“7”，不是“1 + 2 * 3” (就好比C++ 中的constexpr)

### 枚举

枚举可以通过C++注册给Qml使用。5.10以上的版本还可以直接在Qml中定义枚举。

这里分别示例一下：

C++注册枚举给Qml使用
![预览](/images/Qml8/2.png)

C++11的作用域枚举，也是可以的：

![预览](/images/Qml8/3.png)

5.12的版本已经不用写Q_DECLARE_METATYPE(BrotherTao::Country)这一句了，旧一点的版本可能需要写上。

5.10以上版本，Qml中定义枚举:

![预览](/images/Qml8/4.png)

注意 使用枚举必须带上首字母大写的Qml文件名

(这里枚举可能没有语法高亮，但是能正常用，不要担心，那是QtCreator的问题, 可以不管它)

### list

list就是一个列表，但是一般用来存Qml的扩展类型，不能存基础类型。基础类型想要存List，应该用下面的var。

(大部分人用不到这个list，可以跳过)

这里看一下list的用法:

```Qml
Item {
    states: [
          State { name: "activated" },
          State { name: "deactivated" }
    ]
}
```
这种list的实现方式，是在C++中导出了一个特殊类型的属性，即QQmlListProperty。

你也可以自己定义一个这样的属性:

```C++
Q_PROPERTY(QQmlListProperty<Fruit> fruit READ fruit)
```
C++里面按它的规则实现几个函数，并注册类型后，就可以在Qml中这样用:
```Qml
      fruit: [
          Apple {},
          Orange{},
          Banana{}
      ]
```
这个大部分人用不到，就不深入讲解了。

### var

var就相当于js中的var，什么类型都可以存。

```qml
  Item {
      property var aNumber: 100
      property var aBool: false
      property var aString: "Hello world!"
      property var anotherString: String("#FF008800")
      property var aColor: Qt.rgba(0.2, 0.3, 0.4, 0.5)
      property var aRect: Qt.rect(10, 10, 10, 10)
      property var aPoint: Qt.point(10, 10)
      property var aSize: Qt.size(10, 10)
      property var aVector3d: Qt.vector3d(100, 100, 100)
      property var anArray: [1, 2, 3, "four", "five", (function() { return "six"; })]
      property var anObject: { "foo": 10, "bar": 20 }
      property var aFunction: (function() { return "one"; })
  }
```
这种坑人的东西也可以（涛哥我是坚决不会这么用的）:
```qml
  Item {
      property var first:  {}   // nothing = undefined
      property var second: {{}} // empty expression block = undefined
      property var third:  ({}) // empty object
  }
```

C++自定义类型、js的内置类型，都可以用var。你可以在Qt的帮助文档中找到js内置类型:

![预览](/images/Qml8/5.png)

这个文档可能不全面，你可以参考第三方js手册,比如[mozilla-js-reference](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript)

注意，一般将var换成确切的类型会更好一些，Qml引擎处理var有一个转换的过程，会慢一些。

接下来涛哥说几个var的典型用法：

#### var数组

![预览](/images/Qml8/6.png)

如上图，数组的操作基本和js里的Array一致，参考js的手册就行了。

但是要注意的一个问题，无论是改变数组中的单个元素的值，还是增加、删除数组中某个元素，都不会触发数组本身的change信号，绑定其length属性也没有用。

没有change信号，Qml中其它与这个数组关联的地方都没法刷新了。那怎么才能让这个数组刷新呢？

答案是 修改过后，赋值为自己
```qml
    names = names
```
![预览](/images/Qml8/7.png)

原理很简单，拷贝了一个副本，又放回那个地址了，数组本身变了，触发change信号。

(不得不吐槽，js真挫，居然还拷贝了一份。性能肯定好不到哪去)

#### var回调函数

这个用案例来说：

如果你使用过Qml性能探查器(Profiler)，就会发现FileDialog这玩意占很多启动时间、占很多内存。

(涛哥说的是QtQuick.Dialogs里面那个，Qt.labs.platform 那里的实验品从来不用)

好的做法是，Qml工程中只创建一个，复用它。

那么问题来了，某个按钮调用了FileDialog， FileDialog按下确定的时候，怎么把结果传回按钮那里？

这里就要用到回调函数了。按钮调用FileDialog的同时给它一个回调函数，等确定后直接执行回调函数即可。

下面是涛哥封装的TDialog组件，同时支持创建文件、打开文件、打开多个文件、打开文件夹五种用法。
```qml
//TDialog.qml
import QtQuick 2.0
import QtQuick.Dialogs 1.2
Item {
    //顶层使用Item，不用FileDialog，屏蔽FileDialog内部属性和函数
    enum Type {
        CreateFile,
        OpenFile,
        OpenFiles,
        OpenFolder,
    }
    property int __type //参考Qml源码，人为约定 双下划线开头的属性当作私有属性使用，外部不能用。
    
    //点击确定后的回调函数
    property var __acceptCallback: function(file) {}

    FileDialog {
        id: d
        folder: shortcuts.home
        onAccepted: {
            switch(__type) {
            case TDialog.Type.CreateFile:
                __acceptCallback(d.fileUrl)
                break
            case TDialog.Type.OpenFile:
                __acceptCallback(d.fileUrl)
                break
            case TDialog.Type.OpenFiles:
                __acceptCallback(d.fileUrls)
                break
            case TDialog.Type.OpenFolder:
                __acceptCallback(d.folder)
                break
            }
        }
    }
    function createFile(title, nameFilters, callback) {
        __type = TDialog.Type.CreateFile
        d.selectExisting = false
        d.selectFolder = false
        d.selectMultiple = false
        d.title = title
        d.nameFilters = nameFilters
        __acceptCallback = callback
        d.open()
    }
    function openFile(title, nameFilters, callback) {
        __type = TDialog.Type.OpenFile
        d.selectExisting = true
        d.selectFolder = false
        d.selectMultiple = false
        d.title = title
        d.nameFilters = nameFilters
        __acceptCallback = callback
        d.open()
    }
    function openFiles(title, nameFilters, callback) {
        __type = TDialog.Type.OpenFiles
        d.selectExisting = true
        d.selectFolder = false
        d.selectMultiple = true
        d.title = title
        d.nameFilters = nameFilters
        __acceptCallback = callback
        d.open()
    }
    function openFolder(title, callback) {
        __type = TDialog.Type.OpenFolder
        d.selectExisting = true
        d.selectFolder = true
        d.selectMultiple = false
        d.title = title
        __acceptCallback = callback
        d.open()
    }
}
```

下面是使用的示例：
```qml
import QtQuick 2.12
import QtQuick.Window 2.12
import QtQuick.Controls 2.5
Window {
    visible: true
    width: 640
    height: 480
    title: qsTr("Hello Dialog")
    TDialog {
        id: globalDialog
    }

    Row {
        spacing: 10
        Button {
            text: "create file"
            onClicked: {
                globalDialog.createFile("create", ["All files (*.*)"], function(file){
                    console.log("create file", file)
                })
            }
        }
        Button {
            text: "open Image"
            onClicked: {
                globalDialog.openFile("Open one image", ["Image files (*.png *.jpg *.bmp)"], function(file){
                    console.log("Open one image", file)
                })
            }
        }
        Button {
            text: "open Image"
            onClicked: {
                globalDialog.openFiles("Open mulit images", ["Image files (*.png *.jpg *.bmp)"], function(files){
                    console.log("Open mulit images", files)
                })
            }
        }
        Button {
            text: "open folder"
            onClicked: {
                globalDialog.openFolder("Open one folder", function(file){
                    console.log("Open one folder", file)
                })
            }
        }
    }
}

```
TDialog功能收录在最新的[TaoQuick](https://github.com/jaredtao/TaoQuick)项目中，

可以参考源代码，或者到github Release页面下载发布包进行体验。(之前MacOS不能用的问题已经修复)

## Qml模块扩展类型

Qml扩展类型有很多，比如QtQuick模块提供的类型如下：

```
date  Date value
point Value with x and y attributes
rect Value with x, y, width and height attributes
size Value with width and height attributes

```
还有一个非常有用的类型，是QtQml模块提供的Qt：

![预览](/images/Qml8/8.png)

这个Qt自带了很多方法，前面几章提到的Qt.darker和Qt.lighter都是来自这里。

还有这几个也很实用：

Qt.openUrlExternally 可以直接打开一个网址，会自动调用系统的默认浏览器，并跳转到相应的界面

也可以打开一个本地的文件或文件夹，会自动调用系统程序。比如打开一个.txt文件会自动用记事本打开，打开一个文件夹会自动用文件管理器。

Qt.callLater 可以延迟执行，延迟到Qml引擎的事件循环返回(可以用来规避一些隐藏的bug)。

## Qml属性

前面介绍类型的过程中，其实已经用了简单的属性定义。

这里再补充一些前面没有提到的。

### 属性的change信号

写一个普通的属性，隐含的自动生成了一个change信号，信号名字一般是onXxxChanged

```
Item {
    property int value: 12  //这里定义一个属性，并赋初值
    //隐含的已经定义了一个change信号: onValueChanged
}
```
### 属性绑定
```Qml
Item {
    id: root
    property int value1: 12  

    Item {
        property int value2: root.value1 * 10 + 4   //属性初值，依赖另一个属性。这就形成了一种绑定关系
    }
}
```
这里的value2依赖于value1的值，就产生了绑定

所谓的绑定，就是Qml引擎自动做了一个 信号-槽 连接：当onValue1Changed信号发出时，执行 value2 = value1 * 10  + 4。

就是说value1变了，value2会自动跟着变。

### 动态解绑、动态绑定

有时候并不希望两个属性之间，一直是这种绑定状态，需要暂时断开一下绑定关系(解绑)。

这时候只要重新赋值即可解绑，比如
```Qml
Item {
    id: item1
    property int value1: 12  

    Item {
        id: item2
        property int value2: item1.value1 * 10 + 4   //属性初值，依赖另一个属性。这就形成了一种绑定关系
    }
    Button {
        onClicked: {
            item2.value = 1024; //重新赋值，绑定关系被破坏，不会再随着value1的改变而改变。
        }
    }
}

```
解绑后，又需要再次绑定，是不是重新赋值回item1.value1就行了呢？

答案是不对的，再次绑定要用绑定表达式Qt.binding() 
```
Item {
    id: item1
    property int value1: 12  

    Item {
        id: item2
        property int value2: item1.value1 * 10 + 4   //属性初值，依赖另一个属性。这就形成了一种绑定关系
    }
    Button {
        onClicked: {
            item2.value = 1024; //重新赋值，绑定关系被破坏，不会再随着value1的改变而改变。
        }
    }
    Button {
        onClicked: {
            item2.value = item1.value1 * 10 + 4 //这个是赋值表达式，只执行一次，不是绑定表达式。
            item2.value = Qt.binding(function() { return item1.value1 * 10 + 4;}) //这个是绑定表达式。
        }
    }
}

```

### 条件绑定
Qml中还提供了一种功能，叫条件绑定 Binding。前面的动态解绑、再绑定可以用下面的方式实现:

```
Item {
    id: item1
    property int value1: 12  

    Item {
        id: item2
        property int value2
    }
    Binding {
        target: item2
        property: "value2"
        value: item1.value1 * 10 + 4
        when: needBind
    }
    property bool needBind: true
    Button {
        onClicked: {
            needBind = false;   //条件绑定 关闭
            item2.value = 1024; //重新赋值。
        }
    }
    Button {
        onClicked: {
            needBind = true;   //条件绑定打开
        }
    }
}
```

### 只读属性

只读属性就是只能读，不能修改，不产出Change信号。只要在前面写上readonly即可

```qml
    readonly property int maxCPUCount: 8
```

### 默认属性

默认属性一般用在组件封装中,比如要封装一个TaoLabel的组件,这个组件有个Text类型的默认属性叫someText
```qml
// TaoLabel.qml
import QtQuick 2.0
Text {
    default property Text someText
    text: "Hello, " + someText.text
}
```
那么在外面实例化组件的时候，就可以在TaoLabel内部放一个子Text组件，它会自动关联到someText。
```
Item {
    TaoLable {
        Text {text: "world"}
    }
}
```
### 属性别名

别名属性一般用来导出组件内部的属性给外部直接修改

```Qml
Item {
    property alias text: t.text
    Text {
        id: t
    }
}
```
效果与下面的等价，但是别名少一层变量的声明和绑定，效率更高一些。
```Qml
Item {
    id: root
    property string text
    Text {
        id: t
        text: root.text
    }
}
```

### QQmlProperty
 
《Qml组件化编程5-Qml与C++交互》一文中，提到了C++访问Qml的两种方式，这里再补充第三种，就是QQmlProperty

假如有这样的Qml文件，
```Qml
  // MyItem.qml
  import QtQuick 2.0

  Text { text: "A bit of text" }
```
那么在c++种，就可以通过QQmlProperty访问其属性

```C++

  #include <QQmlProperty>
  #include <QGraphicsObject>

  ...

  QQuickView view(QUrl::fromLocalFile("MyItem.qml"));
  QQmlProperty property(view.rootObject(), "font.pixelSize");
  qWarning() << "Current pixel size:" << property.read().toInt();
  property.write(24);
  qWarning() << "Pixel size should now be 24:" << property.read().toInt();

  ```
  QQmlProperty不仅可以用来访问Qml中的属性，还可以调用其信号、js函数。


## 下期预告

下一篇文章，将会介绍C++中的属性导出，重点讲解ListView、TableView、TreeView能用的属性。

敬请期待。