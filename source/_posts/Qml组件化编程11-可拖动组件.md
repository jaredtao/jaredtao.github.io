---
title: Qml组件化编程11-可拖动组件
photos: /img/avatar.jpg
tags:
  - Qt
  - Qt实用技能
  - Qt发布
categories: Qml组件化编程
abbrlink: 210
date: 2019-10-30 18:44:23
---
## 简介

本文是《Qml组件化编程》系列文章的第十一篇，涛哥将教大家，实现Qml可拖动组件。

注：文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)


# 效果图

![1](/images/Qml11/1.gif)

# 代码

[https://github.com/jaredtao/TaoDragBorder](https://github.com/jaredtao/TaoDragBorder)

# 使用

封装的组件名称是TemplateBorder。

使用很简单，在要支持拖拽的目标组件上，创建一个TemplateBorder实例即可，例如：

```qml
    Rectangle {
        x: 100
        y: 200
        width: 300
        height: 200
        color: "red"
        smooth: true
        antialiasing: true
        MouseArea {
            anchors.fill: parent
            onClicked: {
                parent.focus = true
            }
        }
        //这里添加一个边框
        TemplateBorder {
            //注意控制大小
            width: parent.width + borderMargin * 2
            height: parent.height + borderMargin * 2
            anchors.centerIn: parent
            //目标item有焦点时显示边框
            visible: parent.focus
        }
    }
```

# 原理

## 拖拽的前提

目标组件不要用锚布局，不要用Layout布局。

拖拽需要修改目标组件的坐标和宽高，而锚布局、Layout会限定坐标或宽高。

## 拖拽原理

拖拽本身可以使用MouseArea的 drag.target，但这个target限制为item及其子类。

有时候还需要处理无边框窗口的拖动，窗口不是item，就不能用drag.target。

所以需要一个通用的拖拽算法：
```qml
    ...
    MouseArea {
        id: mouseArea
        anchors.fill: parent
        property int lastX: 0
        property int lastY: 0
        onPressedChanged: {
            if (containsPress) {
              //按下鼠标时记录坐标
                lastX = mouseX;
                lastY = mouseY;
            }
        }
        onPositionChanged: {
            if (pressed) {
              //拖动时修改目标的坐标
                parent.x += mouseX - lastX
                parent.y += mouseY - lastY
            }
        }
    }
    ...
```
## 锚点

锚点就是在组件的左上角、右上角等八个点，分别放一个小圆圈，圆圈里面是可拖拽组件，分别控制组件的坐标、宽高。

注意每个点的计算规则都不太一样。

例如左上角，要同时计算x、y和宽高，而右上角则只计算y、和宽高：

```

Item {
    id: root
    //controller 要控制大小的目标，可以是Item，也可以是view，只要提供x、y、width、height等属性的修改
    //默认值为parent
    property var control: parent
    //左上角的拖拽
    TDragItem {
        id: leftTopHandle
        posType: posLeftTop
        onPosChange: {
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
    //右上角拖拽
    TDragItem {
        id: rightTopHandle
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
    ...
    ...
}
```
## 旋转

旋转算法和拖拽类似
```
    MouseArea {
        id: rotateArea
        anchors.centerIn: parent
        property int lastX: 0

        onPressedChanged: {
          if (containsPress) {
            lastX = mouseX;
          }
        }
        onPositionChanged: {
          if (pressed) {
            let t = controller.rotation +(mouseX - lastX) / 5
            //这里的除以5，用来消除抖动。
            t = t % 360
            controller.rotation = t
          }
        }
    }
```