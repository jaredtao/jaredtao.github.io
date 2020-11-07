---
title: Qml特效5-进场动画-百叶窗
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: Qml特效
abbrlink: 51482
date: 2019-06-09 11:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [关于文章](#%E5%85%B3%E4%BA%8E%E6%96%87%E7%AB%A0)
- [百叶窗效果预览](#%E7%99%BE%E5%8F%B6%E7%AA%97%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [实现原理](#%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

## 简介

这是《Qml特效-进场动画》系列文章的第5篇，涛哥将会教大家一些Qml进场动画相关的知识。

进场动画效果 参考了WPS版ppt的动画，基本效果已经全部实现，可以到github TaoQuick项目中预览：

[进场动画预览](https://github.com/jaredtao/TaoQuick/blob/master/Preview-animation.md)

## 关于文章

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 百叶窗效果预览

百叶窗效果，支持向左、向右、向上、向下四个方向

![](/images/Animation/5.gif)

## 实现原理

通过数值动画，控制百分比属性percent从0 到100变化

```
import QtQuick 2.12
import QtQuick.Controls 2.12
ShaderEffect {
    ...
    //枚举声明四种方向
    enum Direct {
        FromLeft = 0,
        FromRight = 1,
        FromTop = 2,
        FromBottom = 3
    }
    property int dir: ASlowEnter.Direct.FromLeft
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
        in vec2 qt_TexCoord0;
        uniform float qt_Opacity;
        uniform sampler2D effectSource;
        uniform int dir;
        uniform int percent;
        uniform int perLen;
        uniform int count;
        out vec4 fragColor;
        void main()
        {
            vec4 color = texture2D(effectSource, qt_TexCoord0);
            int len = 0;
            float coords[] = float[](qt_TexCoord0.x, qt_TexCoord0.y, qt_TexCoord0.x, qt_TexCoord0.y);
            int percents[] = int[](percent, percent, perLen - percent, perLen - percent);
            len = int(coords[dir]* float(perLen) * float(count));
            float alpha =  float(step(percents[dir], mod(len, perLen)));
            float alphas[] = float[](1.0 - alpha, 1.0 - alpha, alpha, alpha);
            alpha = qt_Opacity * alphas[dir];
            fragColor = vec4(color.rgb * alpha, alpha);
        }
```
