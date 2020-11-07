---
title: Qml特效11-进场动画-棋盘
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: Qml特效
abbrlink: 47578
date: 2019-06-09 13:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [关于文章](#%E5%85%B3%E4%BA%8E%E6%96%87%E7%AB%A0)
- [棋盘效果预览](#%E6%A3%8B%E7%9B%98%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [实现原理](#%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

## 简介

这是《Qml特效-进场动画》系列文章的第11篇，涛哥将会教大家一些Qml进场动画相关的知识。

进场动画效果 参考了WPS版ppt的动画，基本效果已经全部实现，可以到github TaoQuick项目中预览：

[进场动画预览](https://github.com/jaredtao/TaoQuick/blob/master/Preview-animation.md)

## 关于文章

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 棋盘效果预览

棋盘效果，支持向右、向下两个方向

![](/images/Animation/11.gif)

## 实现原理

通过数值动画，控制百分比属性percent从0 到100变化

```
import QtQuick 2.12
import QtQuick.Controls 2.12
ShaderEffect {
    ...
    //枚举声明四种方向
    enum Direct {
        ToRight = 0,
        ToBottom = 1
    }
    property int dir : ABoard.Direct.ToRight
    property int percent: 0
    opacity: percent > 0 ? 1 : 0
    NumberAnimation {
        id: animation
        target: r
        property: "percent"
        from: 0
        to: 100
        alwaysRunToEnd: true
        loops: 1
        duration: 1000
    }
    ...
}
```
在Shader中，使用glsl片段着色器实现像素的控制：

```glsl
    fragmentShader: TCommon.fragmentShaderCommon + (dir === ABoard.Direct.ToRight ? "
        in  vec2 qt_TexCoord0;
        uniform float qt_Opacity;
        uniform sampler2D effectSource;
        uniform int dir;
        uniform int percent;
        uniform int rowCount;
        uniform int columnCount;
        out vec4 fragColor;
        void main()
        {
            vec4 color = texture2D(effectSource, qt_TexCoord0);
            int xlen = 100 / columnCount;
            int ylen = 100 / rowCount;
            float p = float(percent) / 100.0;
            int x = int(qt_TexCoord0.x * 100.0);
            int y = int(qt_TexCoord0.y * 100.0);
            float alpha = 0.0;
            int sx = x / xlen;
            int sy = y / ylen;
            int xOffset = 0;
            if (sy % 2 != 0) {
                xOffset = xlen / 2;
            }
            if (sx * xlen <= x + xOffset && (x + xOffset) % xlen <= int(float(xlen) * p) ) {
                alpha = 1.0;
            }
            alpha *= qt_Opacity;
            fragColor = vec4(color.rgb * alpha, alpha);
       }
" : "
        in  vec2 qt_TexCoord0;
        uniform float qt_Opacity;
        uniform sampler2D effectSource;
        uniform int dir;
        uniform int percent;
        uniform int rowCount;
        uniform int columnCount;
        out vec4 fragColor;
        void main()
        {
            vec4 color = texture2D(effectSource, qt_TexCoord0);
            int xlen = 100 / columnCount;
            int ylen = 100 / rowCount;
            float p = float(percent) / 100.0;
            int x = int(qt_TexCoord0.x * 100.0);
            int y = int(qt_TexCoord0.y * 100.0);
            float alpha = 0.0;
            int sx = x / xlen;
            int sy = y / ylen;
            int yOffset = 0;
            if (sx % 2 != 0) {
                yOffset = ylen / 2;
            }
            if (sy * ylen <= y + yOffset && (y + yOffset) % ylen <= int(float(ylen) * p) ) {
                alpha = 1.0;
            }
            alpha *= qt_Opacity;
            fragColor = vec4(color.rgb * alpha, alpha);
       }
")
```
