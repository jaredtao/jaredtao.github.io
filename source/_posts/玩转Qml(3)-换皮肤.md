---
title: 玩转Qml(3)-换皮肤
photos: /images/Qml3/1.webp
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: 玩转Qml
abbrlink: 38586
date: 2019-05-12 12:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [效果预览](#%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [必要的基础](#%E5%BF%85%E8%A6%81%E7%9A%84%E5%9F%BA%E7%A1%80)
  - [QObject自定义属性](#qobject%E8%87%AA%E5%AE%9A%E4%B9%89%E5%B1%9E%E6%80%A7)
  - [全局单例](#%E5%85%A8%E5%B1%80%E5%8D%95%E4%BE%8B)
- [实现](#%E5%AE%9E%E7%8E%B0)
  - [皮肤的配置和原理](#%E7%9A%AE%E8%82%A4%E7%9A%84%E9%85%8D%E7%BD%AE%E5%92%8C%E5%8E%9F%E7%90%86)
  - [皮肤选择器](#%E7%9A%AE%E8%82%A4%E9%80%89%E6%8B%A9%E5%99%A8)
  - [带三角形尖尖的弹窗组件](#%E5%B8%A6%E4%B8%89%E8%A7%92%E5%BD%A2%E5%B0%96%E5%B0%96%E7%9A%84%E5%BC%B9%E7%AA%97%E7%BB%84%E4%BB%B6)


## 简介

本文是《玩转Qml》系列文章的第三篇，涛哥将教大家，如何在Qml中实现动态换皮肤。顺带会分享一些Qt小技巧。

## 源码

《玩转Qml》系列文章，配套了一个优秀的开源项目:TaoQuick

github https://github.com/jaredtao/TaoQuick

访问不了或者速度太慢，可以用国内的镜像网站gitee

https://gitee.com/jaredtao/TaoQuick

## 效果预览

效果类似于网易云音乐

![预览](/images/Qml3/1.webp)

顺便说一下，这是涛哥创建的TaoQuick项目，后续的各种组件、效果会全部集中在这个项目里。

文章中涉及的代码，都会先贴出来。整个工程的代码，在积累到一定程度后，会开放在github上。


## 必要的基础

可能有读者会疑惑，涛哥前两篇文章还在讲如何封装基础组件，这第三篇直接就来换皮肤？是不是跨度有点大？

其实换皮肤是一个很基础的功能，如果要做最好在项目初期就做起来，后期想要做换皮肤会困难一些（工作量大）。

如果你的项目做了很多组件化的封装，再做换皮肤会轻松一些。

### QObject自定义属性

Qml中有一个类型叫QtObject，涛哥非常喜欢使用这个类型。

以前在写Qt/C++代码中的自定义QObject时，经常需要写一些自定义的Q_PROPERTY,以及实现set、get函数、change信号

如果纯手工写，挺累人的，涛哥曾写过自动生成器，相信很多人也写过类似的工具。

后来涛哥发现，QtCreator有自动生成的功能，只要写上Q_PROPERTY那一行，再用右键菜单生成即可，

高效率人士也可以使用快捷键，光标放在Q_PROPERTY上，按Alt + Enter。

![预览](/images/Qml3/2.gif)

有时候需要把set函数的参数改成const T &类型来减少内存拷贝，并把函数实现移动到cpp文件中。(这都是C++的诟病)

然而，在Qml中，有更加方便的QtObject,

```
Item {
    QtObject {  //定义一个QtObject，相当于Qt/c++代码中的QObject
        id: dataObj
        property string name: "Hello"   //这就定义了一个string类型的属性name，初始值设为"Hello"
        
        //定义完了，已经自带了onNameChanged信号。name被重新赋值时会触发信号
        
        //给这个信号写一个处理函数，就相当于连接上了槽函数。
        onNameChanged: {
            console.info("name:", name)
        }
    }
    ...
    Button {
        ...
        onClicked: {    //演示: 按钮按下时，修改前面QtObject的name属性
            dataObj.name = "World"; 
        }
    }
    ...
}
```
(哈哈，工作量和心理负担一下子减轻了很多，头发的数量也能保住了。再也不想回去写Qt/C++的属性了。)

### 全局单例

涛哥写了一个单独的qml文件，顶层就是一个QtObject类型，里面会有一大堆属性。

颜色、字体一类的配置都在这里。

```qml
// GlobalConfig.qml
import QtQuick 2.0
QtObject {
    property color titleBackground: "#c62f2f"   //标题栏的背景色
    property color background: "#f6f6f6"        //标题栏之外的部分的背景色
    property color reserverColor: "#ffffff"     //与背景色相反的对比色
    property color textColor: "black"           //文本颜色
    
}
```
然后在main.qml中实例化它
```qml
// main.qml
Item {
    width: 800
    height: 600
    GlobalConfig {
        id: gConfig
    }
    ...
}
```
Qml有个特性，子页面实例可以通过id访问父页面中的实例，读 写其属性、调用其函数。

在main.qml中实例化的对象，相当于是全局的了，它的id是可以在所有main.qml的子页面中访问到的。

（当然还有一种方式，通过qmldir指定单例，这种留着后面再说）

## 实现

### 皮肤的配置和原理
下面是TaoQuick中使用的gConfig

```qml

// GlobalConfig.qml
QtObject {

    property color titleBackground: "#c62f2f"
    property color background: "#f6f6f6"
    property color reserverColor: "#ffffff"
    property color textColor: "black"
    property color splitColor: "gray"

    property int currentTheme: 0
    onCurrentThemeChanged: {
        var t = themes.get(currentTheme)
        titleBackground = t.titleBackground
        background = t.background
        textColor = t.textColor
    }
    readonly property ListModel themes: ListModel {
        ListElement {
            name: qsTr("一品红")
            titleBackground: "#c62f2f"
            background: "#f6f6f6"
            textColor: "#5c5c5c"
        }
        ListElement {
            name: qsTr("高冷黑")
            titleBackground: "#191b1f"
            background: "#222225"
            textColor: "#adafb2"
        }
        ListElement {
            name: qsTr("淑女粉")
            titleBackground: "#faa0c5"
            background: "#f6f6f6"
            textColor: "#5c5c5c"
        }
        ListElement {
            name: qsTr("富贵金")
            titleBackground: "#fed98f"
            background: "#f6f6f6"
            textColor: "#5c5c5c"
        }
        ListElement {
            name: qsTr(" 清爽绿")
            titleBackground: "#58c979"
            background: "#f6f6f6"
            textColor: "#5c5c5c"
        }
        ListElement {
            name: qsTr("苍穹蓝")
            titleBackground: "#67c1fd"
            background: "#f6f6f6"
            textColor: "#5c5c5c"
        }
    }
}

```

涛哥在所有的Page页面中，相关颜色设置都绑定到gConfig的相应属性上。

那么换皮肤，只需要修改gConfig中的颜色相关属性即可。因为修改属性时会触发change信号，而所有的Page都绑定了

gConfig的属性，会自动在发生change时重新读属性，修改后的颜色自动就生效了。

这里顺带说一下，主题的颜色相关属性越少越好，因为太多了不容易识别、不好维护，

之前的文章《玩转Qml(1)-从按钮开始》中提到的Qt.lighter和Qt.darker,

就是一种减少颜色属性数量的神器。

### 皮肤选择器

再来看一下，涛哥参考 网易云音乐 做的皮肤选择器

```qml
TImageBtn {     //图片按钮，参考文章1
    width: 20
    height: 20
    anchors.verticalCenter: parent.verticalCenter
    imageUrl: containsMouse ? "qrc:/Image/Window/skin_white.png" : "qrc:/Image/Window/skin_gray.png"
    onClicked: {
        skinBox.show()
    }
    TPopup {    //自定义的弹窗，带三角尖尖的那个。
        id: skinBox
        barColor: gConfig.reserverColor
        backgroundWidth: 280
        backgroundHeight: 180
        contentItem: GridView {
            anchors.fill: parent
            anchors.margins: 10
            model: gConfig.themes
            cellWidth: 80
            cellHeight: 80
            delegate: Item {
                width: 80
                height: 80
                Rectangle { //表示主题色的色块
                    anchors.fill: parent
                    anchors.margins: 4
                    height: width
                    color: model.titleBackground
                }
                Rectangle { //主题色边框，鼠标悬浮时显示
                    anchors.fill: parent
                    color: "transparent"
                    border.color: model.titleBackground
                    border.width: 2
                    visible: a.containsMouse
                }
                Text {  //主题名字
                    anchors {
                        left: parent.left
                        bottom: parent.bottom
                        leftMargin: 8
                        bottomMargin: 8
                    }
                    color: "white"
                    text: model.name
                }
                Rectangle { //右下角圆圈圈，当前选中的主题
                    x: parent.width - width
                    y: parent.height - height
                    width: 20
                    height: width
                    radius: width / 2
                    color: model.titleBackground
                    border.width: 3
                    border.color: gConfig.reserverColor
                    visible: gConfig.currentTheme === index
                }
                MouseArea { //鼠标状态
                    id: a
                    anchors.fill: parent
                    hoverEnabled: true
                    onClicked: {    //切主题操作
                        gConfig.currentTheme = index
                    }
                }
            }
        }
        
    }
}
```

### 带三角形尖尖的弹窗组件

```qml
// TPopup.qml
import QtQuick 2.9
import QtQuick.Controls 2.5
Item {
    id: root
    anchors.fill: parent
    property alias popupVisible: popup.visible
    property alias contentItem: popup.contentItem
    property color barColor: "white"
    property alias backgroundItem: background
    property real backgroundWidth: 200
    property real backgroundHeight: 160
    property color borderColor:  barColor
    property real borderWidth: 0

    property real verticalOffset: 20
    //矩形旋转45度，一半被toolTip遮住(重合)，另一半三角形和ToolTip组成一个带箭头的ToolTip
    Rectangle {
        id: bar
        visible: popup.visible
        rotation: 45
        width: 16
        height: 16
        color: barColor
        //水平居中
        anchors.horizontalCenter: parent.horizontalCenter
        //垂直方向上，由ToolTip的y值，决定位置
        anchors.verticalCenter: parent.bottom
        anchors.verticalCenterOffset: verticalOffset
    }
    Popup {
        id: popup
        width: backgroundWidth
        height: backgroundHeight
        background: Rectangle {
            id: background
            color: barColor
            radius: 8
            border.color:borderColor
            border.width: borderWidth
        }
    }
    function show() {
        popup.x = (root.width - popup.width) / 2
        popup.y = root.height + verticalOffset
        popupVisible = true
    }
    function hide() {
        popupVisible = false
    }
}

```