---
title: Qml特效14-跟上节奏
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: Qml特效
abbrlink: 30700
date: 2019-06-11 12:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [关于文章](#%E5%85%B3%E4%BA%8E%E6%96%87%E7%AB%A0)
- [跟上节奏 效果预览](#%E8%B7%9F%E4%B8%8A%E8%8A%82%E5%A5%8F-%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [QtQuick动画系统](#QtQuick%E5%8A%A8%E7%94%BB%E7%B3%BB%E7%BB%9F)
  - [动画组件](#%E5%8A%A8%E7%94%BB%E7%BB%84%E4%BB%B6)
  - [动画的使用](#%E5%8A%A8%E7%94%BB%E7%9A%84%E4%BD%BF%E7%94%A8)
    - [用例一 直接声明动画](#%E7%94%A8%E4%BE%8B%E4%B8%80-%E7%9B%B4%E6%8E%A5%E5%A3%B0%E6%98%8E%E5%8A%A8%E7%94%BB)
    - [用例二 on语法](#%E7%94%A8%E4%BE%8B%E4%BA%8C-on%E8%AF%AD%E6%B3%95)
    - [用例三 Transitions或状态机](#%E7%94%A8%E4%BE%8B%E4%B8%89-Transitions%E6%88%96%E7%8A%B6%E6%80%81%E6%9C%BA)
- [ShaderEffect](#ShaderEffect)
- [跟上节奏 效果源码](#%E8%B7%9F%E4%B8%8A%E8%8A%82%E5%A5%8F-%E6%95%88%E6%9E%9C%E6%BA%90%E7%A0%81)
  - [组件的封装](#%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B0%81%E8%A3%85)
  - [大致原理](#%E5%A4%A7%E8%87%B4%E5%8E%9F%E7%90%86)
  - [组件的使用](#%E7%BB%84%E4%BB%B6%E7%9A%84%E4%BD%BF%E7%94%A8)
  - [倒影效果原理](#%E5%80%92%E5%BD%B1%E6%95%88%E6%9E%9C%E5%8E%9F%E7%90%86)

## 简介

这是《Qml特效》系列文章的第14篇，涛哥将会教大家一些Qml特效和动画相关的知识。

前12篇是进场动画效果 参考了WPS版ppt的动画，12种基本效果已经全部实现，可以到github TaoQuick项目中预览：

[进场动画预览](https://github.com/jaredtao/TaoQuick/blob/master/Preview-animation.md)

因为效果都比较简单，就没有写太多说明。

没有13

没有13

没有13

从14篇开始，做一些特殊的效果和动画，同时会讲解相关的知识。

## 关于文章

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 跟上节奏 效果预览

跳动的字节，以及倒影

![](/images/Ani1/Soundbyte.gif)

其中用到的知识点有: Qml动态创建组件、Qml随机数生成、Qml给Item拍照(layer属性)、Qml翻转。

当然还有一些预备知识：

做特效和动画，要用到QtQuick的动画系统，以及ShaderEffect特效。

## QtQuick动画系统

### 动画组件

Qt动画系统，在帮助文档有详细的介绍，搜索关键词"Animation"，涛哥在这里说一些重点。

涛哥用思维导图列出了Qml中所有的动画组件:

![](/images/Ani1/1.png)

* 右边带虚线框的部分比较常用，是做动画必须要掌握的，尤其是属性动画PropertyAnimation和数值动画NumberAinmation。
常见的各种坐标动画、宽高动画、透明度动画、颜色动画等等，都可以用这些组件来实现。

* 底下的States、Behavior 和 Traisitions，也是比较常用的和动画相关的组件。可在帮助文档搜索
关键词"Qt Quick States"、"Behavior"、"Animation and Transitions"。后续的文章，涛哥会专门讲解。

* 左边的Animator系列，属于Scene Graph渲染层面的优化，其属性Change信号只在最终值时发出，不发出中间值，使用的时候需要注意。

* 顶上的AnimationController，属于高端玩家，用来控制整个动画的进度。

### 动画的使用

#### 用例一 直接声明动画 

直接声明动画，指定target和property，之后可以在槽函数/js脚本中通过id控制动画的运行。

也可以通过设定loops 和 running属性来控制动画

```qml
  Rectangle {
      id: flashingblob
      width: 75; height: 75
      color: "blue"
      opacity: 1.0

      MouseArea {
          anchors.fill: parent
          onClicked: {
              animateColor.start()
              animateOpacity.start()
          }
      }

      PropertyAnimation {id: animateColor; target: flashingblob; properties: "color"; to: "green"; duration: 100}

      NumberAnimation {
          id: animateOpacity
          target: flashingblob
          properties: "opacity"
          from: 0.99
          to: 1.0
          loops: Animation.Infinite
          easing {type: Easing.OutBack; overshoot: 500}
     }
  }
```

#### 用例二 on语法 

on语法可以使用动画组件，也可以用Behavior，直接on某个特定的属性即可。效果一样。

on动画中，如果直接指定了running属性，默认就会执行这个动画。

也可以不指定running属性，其它地方修改这个属性时，会自动按照动画来执行。

示例代码 on动画
```
  Rectangle {
      width: 100; height: 100; color: "green"
      RotationAnimation on rotation {
          loops: Animation.Infinite
          from: 0
          to: 360
          running: true
      }
  }
```

示例代码 Behavior 动画
```
  import QtQuick 2.0

  Rectangle {
      id: rect
      width: 100; height: 100
      color: "red"

      Behavior on width {
          NumberAnimation { duration: 1000 }
      }

      MouseArea {
          anchors.fill: parent
          onClicked: rect.width = 50
      }
  }
```

#### 用例三 Transitions或状态机

过渡动画和状态机动画，本质还是直接使用动画组件。只不过是把动画声明并存储起来，以在状态切换时使用。

这里先不细说了，后面会有系列文章<Qml特效-页面切换动画>，会专门讲解。

## ShaderEffect

动画只能控制组件的属性整体的变化，做特效需要精确到像素。

Qml中提供了ShaderEffect这个组件，就能实现像素级别的操作。

Qml中有一个模块QtGraphicalEffects，提供了部分特效，就是使用ShaderEffect实现的。

使用ShaderEffect实现特效，需要有一些OpenGL/DirectX知识，了解GPU渲染管线，同时也需要一些数学知识。

大名鼎鼎的ShaderToy网站，就是使用Shader实现各种像素级别的酷炫特效。

[ShaderToy](https://www.shadertoy.com)

[作者iq大神](http://www.iquilezles.org/www/articles/raymarchingdf/raymarchingdf.htm)

ShaderToy上面的特效都是可以移植到Qml中的。

使用Shader开发，需要一定的图形学知识。其中使用GLSL需要熟悉OpenGL, 使用HLSL需要熟悉DirectX。

## 跟上节奏 效果源码

### 组件的封装

封装了一个组件TSoundByte

```qml
//TSoundByte.qml
import QtQuick 2.12

Item {
    id: r
    implicitWidth: (soundWidth + columnSpacing) * columnCount
    implicitHeight: (soundHeight + rowSpacing) * rowCount
    Component {
        id: soundComp
        Image {
            source: "qrc:/EffectImage/Img/soundRect.png"
            width: soundWidth
            height: soundHeight
        }
    }
    property bool running: true
    property int interval: 320
    property int soundWidth: 12
    property int soundHeight: 6

    property int rowCount: 15
    property int columnCount: 30

    property int rowSpacing: 4
    property int columnSpacing: 4

    property var objPool: []
    property var map:[]
    property int __arrayRatio: 100
    Component.onCompleted: {
        let startX = 0
        let startY = r.height - 12
        for (let i = 0; i < columnCount; ++i) {
            map.push(getRandomInt(0, rowCount))

            let px = startX + i * (soundWidth + columnSpacing)
            for (let j = 0; j < rowCount; ++j) {
                let py =  startY - j * (soundHeight + rowSpacing)

                var obj = soundComp.createObject(r, {"x": px, "y": py, "visible": false})
                objPool[i * __arrayRatio + j] = obj
            }
        }
    }
    Timer {
        interval: r.interval
        running: r.running
        repeat: true
        onTriggered: {
            map.push(getRandomInt(0, rowCount))
            map.shift()
            for (let i = 0; i < columnCount; ++i) {
                for (let j = 0; j < rowCount; ++j) {
                    objPool[i * __arrayRatio + j]["visible"] = j < map[i] ? true : false
                }
            }
        }
    }
    function getRandomInt(min, max) {
      min = Math.ceil(min);
      max = Math.floor(max);
      return Math.floor(Math.random() * (max - min)) + min; //不含最大值，含最小值
    }
}

```

### 大致原理

这里的soundComp就是一个Component组件，在槽函数/js函数中，可以用id去动态创建。

soundComp.createObject(r, {"x": px, "y": py, "visible": false})

涛哥一次性创建了n * m 个Component，都设置好相对的坐标，存储到objPool数组中。

visible属性根据预定义的数组map来控制。

之后在定时器中，用一个随机高度去刷新map。然后再更新objPool中的visible即可。

### 组件的使用

```qml
import QtQuick 2.12
import QtQuick.Controls 2.12
import "../Effects"
Rectangle  {
    anchors.fill: parent
    color: "black"
    TSoundByte {
        id: src
        anchors.centerIn: parent
        anchors.verticalCenterOffset: -100
        interval: 240
    }

    ShaderEffectSource {
        id: mirror
        width: src.width
        height: src.height
        x: src.x
        y: src.y + src.height + 10
        opacity: 0.3
        sourceItem: src
        transform: Rotation {
            origin.x: mirror.width/2
            origin.y: mirror.height/2
            axis.x: 1; axis.y: 0; axis.z: 0
            angle: 180
        }
        //no effect
        //textureMirroring: ShaderEffectSource.MirrorHorizontally

    }
}

```

### 倒影效果原理

先是实例化了一个TSoundByte组件，id为src，之后又在实例src的坐标下面，放了一个ShaderEffectSource，指定其sourceItem为src。

这样就有了src的一个复制品mirror。然后把mirror翻转180度，透明度调一下，就变成倒影效果了。