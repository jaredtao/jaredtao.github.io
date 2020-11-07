---
title: Qml特效15-暗流涌动
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: Qml特效
abbrlink: 14201
date: 2019-06-11 13:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [关于文章](#%E5%85%B3%E4%BA%8E%E6%96%87%E7%AB%A0)
- [暗流涌动 效果预览](#%E6%9A%97%E6%B5%81%E6%B6%8C%E5%8A%A8-%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [QtQuick动画系统](#QtQuick%E5%8A%A8%E7%94%BB%E7%B3%BB%E7%BB%9F)
  - [动画组件](#%E5%8A%A8%E7%94%BB%E7%BB%84%E4%BB%B6)
  - [动画的使用](#%E5%8A%A8%E7%94%BB%E7%9A%84%E4%BD%BF%E7%94%A8)
    - [用例一 直接声明动画](#%E7%94%A8%E4%BE%8B%E4%B8%80-%E7%9B%B4%E6%8E%A5%E5%A3%B0%E6%98%8E%E5%8A%A8%E7%94%BB)
    - [用例二 on语法](#%E7%94%A8%E4%BE%8B%E4%BA%8C-on%E8%AF%AD%E6%B3%95)
    - [用例三 Transitions或状态机](#%E7%94%A8%E4%BE%8B%E4%B8%89-Transitions%E6%88%96%E7%8A%B6%E6%80%81%E6%9C%BA)
- [ShaderEffect](#ShaderEffect)
- [暗流涌动 效果源码](#%E6%9A%97%E6%B5%81%E6%B6%8C%E5%8A%A8-%E6%95%88%E6%9E%9C%E6%BA%90%E7%A0%81)
  - [组件的封装](#%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B0%81%E8%A3%85)
  - [原理](#%E5%8E%9F%E7%90%86)
  - [组件的使用](#%E7%BB%84%E4%BB%B6%E7%9A%84%E4%BD%BF%E7%94%A8)

## 简介

这是《Qml特效》系列文章的第15篇，涛哥将会教大家一些Qml特效和动画相关的知识。

前12篇是进场动画效果 参考了WPS版ppt的动画，12种基本效果已经全部实现，可以到github TaoQuick项目中预览：

[进场动画预览](https://github.com/jaredtao/TaoQuick/blob/master/Preview-animation.md)

因为效果都比较简单，就没有写太多说明。

没有13

没有13

没有13

从14篇开始，做一些特殊的效果和动画，同时会讲解相关的知识。

## 关于文章

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 暗流涌动 效果预览

流动的箭头，以及倒影

![](/images/Ani1/Arrow.gif)

其中用到的知识点有: Qml路径动画。

当然还有一些预备知识：

做特效和动画，要用到QtQuick的动画系统，以及ShaderEffect特效。

## 暗流涌动 效果源码

### 组件的封装

封装了一个组件TArrow

```qml
//TArrow.qml
import QtQuick 2.12
import QtQuick.Controls 2.12
Image {
    id: root
    x: 10
    y: 10
    source: "qrc:/EffectImage/Img/arrow.png"
    visible: false
    function run() {
        visible = true;
        pathAnimation.start();
    }
    PathAnimation {
        id: pathAnimation
        target: root
        loops: -1
        duration: 2400
        orientation: PathAnimation.TopFirst
        path: Path{
            startX: 10
            startY: 10
            PathCurve { x: 60; y: 15}
            PathCurve { x: 110; y: 205}
            PathCurve { x: 210; y: 200}
            PathCurve { x: 260; y: 35}
            PathCurve { x: 310; y: 25}
        }
    }
}


```

### 原理

很简单的路径动画,使用的箭头图片是这张:

![](/images/Ani1/arrow.png)

### 组件的使用

```qml
import QtQuick 2.12
import QtQuick.Controls 2.12
import "../Effects/"
Rectangle {
    anchors.fill: parent
    color: "black"
    Item {
        id: arrowItem
        x: 10
        y: 10
        width: 300
        height: 300
        TArrow {
            id: arrow1
        }
        TArrow {
            id: arrow2
        }
        TArrow {
            id: arrow3
        }
        TArrow {
            id: arrow4
        }
        TArrow {
            id: arrow5
        }
        TArrow {
            id: arrow6
        }
        TArrow {
            id: arrow7
        }
        TArrow {
            id: arrow8
        }
        TArrow {
            id: arrow9
        }
    }
    Item {
        id: mirrorItem
        x: arrowItem.x
        y: arrowItem.y + arrowItem.height
        width: arrowItem.width
        height: arrowItem.height
        opacity: 0.3
        layer.enabled: true
        layer.effect: Component {
            ShaderEffectSource {
                sourceItem: arrowItem
                textureMirroring: ShaderEffectSource.MirrorVertically
            }
        }
        transform: Rotation {
            origin.x: mirrorItem.width / 2
            origin.y: mirrorItem.height / 2
            axis {x: 1; y: 0; z: 0}
            angle: 180
        }
    }
    Component.onCompleted: {
        seAnimation.start()
    }
    SequentialAnimation {
        id: seAnimation
        ScriptAction {script: arrow1.run()}
        PauseAnimation {duration: 200 }
        ScriptAction {script: arrow2.run()}
        PauseAnimation {duration: 200 }
        ScriptAction {script: arrow3.run()}

        PauseAnimation {duration: 500 }

        ScriptAction {script: arrow4.run()}
        PauseAnimation {duration: 200 }
        ScriptAction {script: arrow5.run()}
        PauseAnimation {duration: 200 }
        ScriptAction {script: arrow6.run()}

        PauseAnimation {duration: 500 }

        ScriptAction {script: arrow7.run()}
        PauseAnimation {duration: 200 }
        ScriptAction {script: arrow8.run()}
        PauseAnimation {duration: 200 }
        ScriptAction {script: arrow9.run()}
        PauseAnimation {duration: 200 }
    }
}


```

动画比较简单粗暴。

倒影这里，和《Qml特效14-跟上节奏》使用的不一样，使用的是Item的layer属性

layer.enabled: true

layer.effect: Component {
    ShaderEffectSource {
        
    }
}