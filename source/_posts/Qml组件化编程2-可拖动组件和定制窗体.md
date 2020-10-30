---
title: Qml组件化编程2-可拖动组件和定制窗体
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: Qml组件化编程
abbrlink: 10619
date: 2019-05-11 12:22:33
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [拖动组件](#%E6%8B%96%E5%8A%A8%E7%BB%84%E4%BB%B6)
  - [拖动改变坐标](#%E6%8B%96%E5%8A%A8%E6%94%B9%E5%8F%98%E5%9D%90%E6%A0%87)
  - [拖动改变大小](#%E6%8B%96%E5%8A%A8%E6%94%B9%E5%8F%98%E5%A4%A7%E5%B0%8F)
  - [融合](#%E8%9E%8D%E5%90%88)
  - [多级组件和Qml应用的框架结构](#%E5%A4%9A%E7%BA%A7%E7%BB%84%E4%BB%B6%E5%92%8Cqml%E5%BA%94%E7%94%A8%E7%9A%84%E6%A1%86%E6%9E%B6%E7%BB%93%E6%9E%84)
- [自定义窗口](#%E8%87%AA%E5%AE%9A%E4%B9%89%E7%AA%97%E5%8F%A3)
  - [无边框](#%E6%97%A0%E8%BE%B9%E6%A1%86)
  - [可拖动窗口](#%E5%8F%AF%E6%8B%96%E5%8A%A8%E7%AA%97%E5%8F%A3)
  - [自定义标题栏](#%E8%87%AA%E5%AE%9A%E4%B9%89%E6%A0%87%E9%A2%98%E6%A0%8F)
- [效果](#%E6%95%88%E6%9E%9C)

## 简介

本文是《Qml组件化编程》系列文章的第二篇，涛哥将教大家，如何在Qml中实现可拖动组件，通过拖动
 
改变组件的大小和位置；以及实现定制窗体（无边框和标题栏）, 并把拖动组件应用在顶层窗体。

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [知乎专栏-涛哥的Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)。

## 拖动组件

### 拖动改变坐标

 拖动改变坐标的原理很简单，鼠标移动的时候改变目标Item的坐标即可。

 说话的功夫，涛哥就造了个轮子出来
 
 (其实是太常用了，涛哥已经写了很多遍)

 ```qml
import QtQuick 2.9
import QtQuick.Controls 2.5
Item {
    width: 800
    height: 600

    Rectangle {
        id: moveItem

        //注意拖动目标不要使用锚布局或者Layout，而是使用相对坐标
        x: 100
        y: 100
        width: 300
        height: 200

        color: "lightblue"
        MouseArea {
            anchors.fill: parent
            property real lastX: 0
            property real lastY: 0
            onPressed: {
                //鼠标按下时，记录鼠标初始位置
                lastX = mouseX
                lastY = mouseY
            }
            onPositionChanged: {
                if (pressed) { 
                    //鼠标按住的前提下，坐标改变时，计算偏移量，应用到目标item的坐标上即可
                    moveItem.x += mouseX - lastX
                    moveItem.y += mouseY - lastY
                }
            }
        }
    }
}
 ```
![预览](/images/Qml2/1.gif)

上面例子中的MouseArea是拖动区域，Rectangle是要拖动的目标Item。

为了实现高度的可复用性，涛哥将MouseArea独立封装成一个组件,并提供一个control属性，

让外部使用组件实例的时候指定要拖动的目标。

```qml
// TMoveArea.qml

import QtQuick 2.9

MouseArea {
    id: root

    property real lastX: 0
    property real lastY: 0
    property bool mask: false       //有时候外面需要屏蔽拖动，导出一个mask属性， 默认false。
    property var control: parent   //导出一个control属性，指定要拖动的目标， 默认就用parent好了。注意目标要有x和y属性并且可修改

    onPressed: {
        lastX = mouseX;
        lastY = mouseY;
    }
    onContainsMouseChanged: { //修改一下鼠标样式，以示区别
        if (containsMouse) {
            cursorShape = Qt.SizeAllCursor;
        } else {
            cursorShape = Qt.ArrowCursor;
        }
    }
    onPositionChanged: {
        if (!mask && pressed && control)
        {
            control.x +=mouseX - lastX
            control.y +=mouseY - lastY
        }
    }
}

```

TMoveArea组件的用法

```qml
Item {
    anchors.fill: parent

    Rectangle {
        x: 100
        y: 200
        width: 400
        height: 300
        color: "darkred"
        //实例化一个MoveArea
        TMoveArea {
            //指定control为parent。 其实默认就是parent，写出来示意一下
            control: parent
            anchors.fill: parent
        }
    }
}

```

一般来说，将

```qml
  property var control: parent
```

中的var换成确切的类型比如Item会更好一些，Qml底层引擎处理var会慢一些，但是这样就限制了

目标必须是Item或者其子类。var是把双刃剑，有利有弊。涛哥后面要拖动的目标还包括QQuickView

这种类型，所以这里用var就好了。

### 拖动改变大小

拖动改变大小，原理参考下面这张示意图：

![预览](/images/Qml2/2.png)

就是在要拖动的目标Item的8个位置分别放一个拖动组件，并在拖动时计算相应的坐标和大小变化即可。

涛哥先是把TMoveArea改造成了TDragRect

```qml
// TDragRect.qml
import QtQuick 2.9
import QtQuick.Controls 2.0
Item {
    id: root
    property alias containsMouse: mouseArea.containsMouse
    signal posChange(int xOffset, int yOffset)
    implicitWidth: 12   //这里隐式的宽为12
    implicitHeight: 12  //这里隐式的高为12
    property int posType: Qt.ArrowCursor

    //5.10之前, qml是不能定义枚举的，用只读的int属性代替一下。
    readonly property int posLeftTop: Qt.SizeFDiagCursor
    readonly property int posLeft: Qt.SizeHorCursor
    readonly property int posLeftBottom: Qt.SizeBDiagCursor
    readonly property int posTop: Qt.SizeVerCursor
    readonly property int posBottom: Qt.SizeVerCursor
    readonly property int posRightTop: Qt.SizeBDiagCursor
    readonly property int posRight: Qt.SizeHorCursor
    readonly property int posRightBottom: Qt.SizeFDiagCursor
    MouseArea {
        id: mouseArea
        anchors.fill: parent
        hoverEnabled: true
        property int lastX: 0
        property int lastY: 0
        onContainsMouseChanged: {
            if (containsMouse) {
                cursorShape = posType;
            } else {
                cursorShape = Qt.ArrowCursor;
            }
        }
        onPressedChanged: {
            if (containsPress) {
                lastX = mouseX;
                lastY = mouseY;
            }
        }
        onPositionChanged: {
            if (pressed) {
                posChange(mouseX - lastX, mouseY - lastY)
            }
        }
    }
}

```

就是把前面的鼠标拖动时的处理逻辑，换成了带参数的信号发送出去，由外面决定怎么用这两个坐标

同时也定义了一组枚举，用来表示拖动区域的位置。位置不同，则鼠标样式不同。

之后涛哥写了一个叫TResizeBorder的组件，里面实例化了8个TDragRect组件，分别放在前面示意图

所示的位置，并实现了不同的处理逻辑。

(后来涛哥把上下左右四个中心点换成了四个边)

```qml
// TResizeBorder.qml
import QtQuick 2.7

Rectangle {
    id: root
    color: "transparent"
    border.width: 4
    border.color: "black"
    width: parent.width
    height: parent.height
    property var control: parent
    TDragRect {
        posType: posLeftTop
        onPosChange: {
            //不要简化这个判断条件，至少让以后维护的人能看懂。化简过后我自己都看不懂了。
            if (control.x + xOffset < control.x + control.width)
                control.x += xOffset;
            if (control.y + yOffset < control.y + control.height)
                control.y += yOffset;
            if (control.width - xOffset > 0)
                control.width-= xOffset;
            if (control.height -yOffset > 0)
                control.height -= yOffset;
        }
    }
    TDragRect {
        posType: posMidTop
        x: (parent.width - width) / 2
        onPosChange: {
            if (control.y + yOffset < control.y + control.height)
                control.y += yOffset;
            if (control.height - yOffset > 0)
                control.height -= yOffset;
        }
    }
    TDragRect {
        posType: posRightTop
        x: parent.width - width
        onPosChange: {
            //向左拖动时，xOffset为负数
            if (control.width + xOffset > 0)
                control.width += xOffset;
            if (control.height - yOffset > 0)
                control.height -= yOffset;
            if (control.y + yOffset < control.y + control.height)
                control.y += yOffset;
        }
    }
    TDragRect {
        posType: posLeftMid
        y: (parent.height - height) / 2
        onPosChange: {
            if (control.x + xOffset < control.x + control.width)
                control.x += xOffset;
            if (control.width - xOffset > 0)
                control.width-= xOffset;
        }
    }
    TDragRect {
        posType: posRightMid
        x: parent.width - width
        y: (parent.height - height) / 2
        onPosChange: {
            if (control.width + xOffset > 0)
                control.width += xOffset;
        }
    }
    TDragRect {
        posType: posLeftBottom
        y: parent.height - height
        onPosChange: {
            if (control.x + xOffset < control.x + control.width)
                control.x += xOffset;
            if (control.width - xOffset > 0)
                control.width-= xOffset;
            if (control.height + yOffset > 0)
                control.height += yOffset;
        }
    }
    TDragRect {
        posType: posMidBottom
        x: (parent.width - width) / 2
        y: parent.height - height
        onPosChange: {
            if (control.height + yOffset > 0)
                control.height += yOffset;
        }
    }
    TDragRect {
        posType: posRightBottom
        x: parent.width - width
        y: parent.height - height
        onPosChange: {
            if (control.width + xOffset > 0)
                control.width += xOffset;
            if (control.height + yOffset > 0)
                control.height += yOffset;
        }
    }
}

```

注意组件的顶层，使用的是透明的Rectangle，这样做的目的是，外面可以给这个组件设置

不同的颜色、边框等。无论哪种UI框架，透明处理都是需要一定的性能消耗的，所以在不需要显示

出来的情况下，组件顶层最好还是用Item替代。

### 融合

我们来实例化一个能拖动改变大小和位置的Item

```qml
Item {
    width: 800
    height: 600
    Rectangle {
        x: 300
        y: 200
        width: 120
        height: 80
        color: "darkred"
        TMoveArea {
            anchors.fill: parent
            control: parent     //默认就是parent，可以不写。这里写出来示意一下。
        }
        TResizeBorder {
            control: parent     //默认就是parent，可以不写。这里写出来示意一下。
            anchors.fill: parent

        }
    }
}
```

![预览](/images/Qml2/3.webp)

用起来还是挺方便的，直接在目标Item里面实例化TMoveArea和TResizeBorder两个组件，作为目标Item的child，

分别指定control,就把两种功能 融合起来了。注意前后顺序，如果反过来写则TMoveArea会把TResizeBorder遮

盖住。（Qml是有z轴的，以后的文章涛哥再讲）

### 多级组件和Qml应用的框架结构

回过头来看一下，先是封装了两个组件：TMoveArea和TDragRect，之后又封装了一个组件：TResizeBorder，

而这个TResizeBorder里面使用了多个TDragRect组件，显然是有层级结构在里面的。

涛哥把TMoveArea和TDragRect这样的最基础的组件叫做一级组件，那么TResizeBorder就是一个二级组件。

涛哥大量的实战经验后，总结出了这样一种Qml应用框架结构：

    一级和二级组件可以单独做成一个插件(或者叫Qml通用库)。

    实际的Qml项目，在这些基础上，做一些功能性或者业务性的组件，即三级组件。

    由这些三级组件组成一堆的页面(Page)。

    最终的main.qml中，只剩下Page的布局。


示意图如下：


![预览](/images/Qml2/4.png)

## 自定义窗口

自定义窗口，这里以QQuickView为主。

### 无边框

去掉边框，需要在C++中设置flag为Qt::FramelessWindowHint

同时我们注册view到qml上下文环境，给后面的功能来使用。

```c++
    ...
    QQuickView view;
    view.setFlag(Qt::FramelessWindowHint);
    view.rootContext()->setContextProperty("view", &view);
    ...
```

### 可拖动窗口

将我们前面做的两种拖动框放在main.qml中，填满顶层Item，并指定control为view。

```qml
//main.qml

import QtQuick 2.0

#import TaoQuick 1.0      //这里是做成插件的情况下，引用了插件
#import "qrc:/Tao/Qml"    //没有做插件的情况下，只要引用qml文件的资源路径即可

Item {
    //标题栏
    TitlePage {
        id: titleRect
        width: root.width
        height: 60
        ...
        //标题栏区域，实例化一个可以拖动位置的组件
        TMoveArea {
            height: parent.height
            anchors {
                left: parent.left
                right: parent.right
                rightMargin: 170 //留一点右边距，给最小化、最大化、关闭等按钮用
            }
            //指定拖动目标为view
            control: view
        }
        ...
    }
    //实例化一个拖动改大小的组件
    TResizeBorder {
        //指定拖动目标为view
        control: view
        anchors.fill: parent
    }
    ...
}

```

### 自定义标题栏

标题栏的关键就是实现右侧的三个按钮，如果你看了[《Qml组件化编程1-按钮的定制与封装》](https://jaredtao.github.io/2019/05/09/Qml%E7%BB%84%E4%BB%B6%E5%8C%96%E7%BC%96%E7%A8%8B1-%E6%8C%89%E9%92%AE%E7%9A%84%E5%AE%9A%E5%88%B6%E4%B8%8E%E5%B0%81%E8%A3%85/)，

这都没有什么难度了。涛哥这里用图片按钮的方式实现。

注意最大化按钮在最大化状态下变成标准化按钮。

最小化：view.showMinimized()

最大化：view.showMaximized()

标准化：view.showNormal()

关闭: view.close()

这里给出关键代码

```qml
Item{
    ...
    property bool isMaxed: false
    Row {
        id: controlButtons
        height: 20
        anchors.verticalCenter: parent.verticalCenter
        anchors.right: parent.right
        anchors.rightMargin: 12
        spacing: 10
        TImageBtn {
            width: 20
            height: 20
            imageUrl: containsMouse ? "qrc:/Image/Window/minimal_white.png" : "qrc:/Image/Window/minimal_gray.png"
            onClicked: {
                view.showMinimized()
            }
        }
        TImageBtn {
            width: 20
            height: 20
            visible: !isMaxed
            imageUrl: containsMouse ? "qrc:/Image/Window/max_white.png" : "qrc:/Image/Window/max_gray.png"
            onClicked: {
                view.showMaximized()
                isMaxed = true
            }
        }
        TImageBtn {
            width: 20
            height: 20
            visible: isMaxed
            imageUrl: containsMouse ? "qrc:/Image/Window/normal_white.png" : "qrc:/Image/Window/normal_gray.png"
            onClicked: {
                view.showNormal()
                isMaxed = false
            }
        }
        TImageBtn {
            width: 20
            height: 20
            imageUrl: containsMouse ? "qrc:/Image/Window/close_white.png" : "qrc:/Image/Window/close_gray.png"
            onClicked: {
                view.close()
            }
        }
    }
}
```

## 效果

最后，我们来看一下效果吧

![预览](/images/Qml2/5.gif)
