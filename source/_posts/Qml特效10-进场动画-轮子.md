---
title: Qml特效10-进场动画-轮子
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: Qml特效
abbrlink: 61151
date: 2019-06-09 13:04:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [关于文章](#%E5%85%B3%E4%BA%8E%E6%96%87%E7%AB%A0)
- [轮子效果预览](#%E8%BD%AE%E5%AD%90%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [实现原理](#%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

## 简介

这是《Qml特效-进场动画》系列文章的第10篇，涛哥将会教大家一些Qml进场动画相关的知识。

进场动画效果 参考了WPS版ppt的动画，基本效果已经全部实现，可以到github TaoQuick项目中预览：

[进场动画预览](https://github.com/jaredtao/TaoQuick/blob/master/Preview-animation.md)

## 关于文章

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 轮子效果预览

轮子效果，支持顺时针、逆时针两个反向，以及扇形方向

![](/images/Animation/10.gif)

## 实现原理

通过数值动画，控制百分比属性percent从0 到100变化

```
import QtQuick 2.12
import QtQuick.Controls 2.12
ShaderEffect {
    ...
    enum Direct {
        Clockwise = 0,
        CounterClockwise = 1
    }
    property int dir : AWheel.Direct.Clockwise
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
fragmentShader: TCommon.fragmentShaderCommon + (dir === AWheel.Direct.CounterClockwise ? "
        in vec2 qt_TexCoord0;
        uniform float qt_Opacity;
        uniform sampler2D effectSource;
        uniform int dir;
        uniform int percent;
        out vec4 fragColor;
        void main()
        {
            vec4 color = texture2D(effectSource, qt_TexCoord0);
            float per = float(percent) / 100.0 * -360.0;
            const vec2 origin = vec2(0.5, 0.5);
            const vec2 a = vec2(0.5, 0.0);
            vec2 c = qt_TexCoord0 - origin;
            float alpha = 0.0;
            if (per >= -180.0) {
                if (qt_TexCoord0.y > 0.5) {
                    if (cos(radians(per))* length(c) * length(a) < dot(c, a)) {
                        alpha = 1.0;
                    }
                }
            } else {
                if (qt_TexCoord0.y < 0.5) {
                    if (cos(radians(per))* length(c) * length(a) > dot(c, a)) {
                        alpha = 1.0;
                    }
                } else {
                    alpha = 1.0;
                }
            }
            alpha *= qt_Opacity;
            fragColor = vec4(color.rgb * alpha, alpha);
       }
" : "
        in vec2 qt_TexCoord0;
        uniform float qt_Opacity;
        uniform sampler2D effectSource;
        uniform int dir;
        uniform int percent;
        out vec4 fragColor;
        void main()
        {
            vec4 color = texture2D(effectSource, qt_TexCoord0);
            float per = float(percent) / 100.0 * 360.0;
            const vec2 origin = vec2(0.5, 0.5);
            const vec2 a = vec2(0.5, 0.0);
            vec2 c = qt_TexCoord0 - origin;
            float alpha = 0.0;
            if (per <= 180.0) {
                if (qt_TexCoord0.y < 0.5) {
                    if (cos(radians(per))* length(c) * length(a) < dot(c, a)) {
                        alpha = 1.0;
                    }
                }
            } else {
                if (qt_TexCoord0.y > 0.5) {
                    if (cos(radians(per))* length(c) * length(a) > dot(c, a)) {
                        alpha = 1.0;
                    }
                } else {
                    alpha = 1.0;
                }
            }
            alpha *= qt_Opacity;
            fragColor = vec4(color.rgb * alpha, alpha);
       }
")
```
扇形效果：
```glsl
        in vec2 qt_TexCoord0;
        uniform float qt_Opacity;
        uniform sampler2D effectSource;
        uniform int percent;
        out vec4 fragColor;
        void main()
        {
            vec4 color = texture2D(effectSource, qt_TexCoord0);
            float per = float(percent) / 100.0 * 180;
            const vec2 origin = vec2(0.5, 0.5);
            const vec2 a = vec2(0.5, 0.0);
            vec2 c = qt_TexCoord0 - origin;
            float alpha = 0.0;
            if (cos(radians(per))* length(c) * length(a) < dot(c, a)) {
                alpha = 1.0;
            }
            alpha *= qt_Opacity;
            fragColor = vec4(color.rgb * alpha, alpha);
       }

```