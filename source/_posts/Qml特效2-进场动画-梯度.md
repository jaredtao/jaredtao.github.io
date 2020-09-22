---
title: Qml特效2-进场动画-梯度
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: Qml特效
abbrlink: 19587
date: 2019-06-09 09:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [关于文章](#%E5%85%B3%E4%BA%8E%E6%96%87%E7%AB%A0)
- [梯度效果预览](#%E6%A2%AF%E5%BA%A6%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [实现原理](#%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)

## 简介

这是《Qml特效-进场动画》系列文章的第二篇，涛哥将会教大家一些Qml进场动画相关的知识。

进场动画效果 参考了WPS版ppt的动画，基本效果已经全部实现，可以到github TaoQuick项目中预览：

[进场动画预览](https://github.com/jaredtao/TaoQuick/blob/master/Preview-animation.md)

## 关于文章

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 梯度效果预览

梯度效果，支持从四个方向梯度出现

![](/images/Animation/2.gif)

## 实现原理

通过数值动画，控制百分比属性percent从0 到100变化

```
//AGrad.qml
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
        out vec4 fragColor;
        void main()
        {
            vec4 color = texture2D(effectSource, qt_TexCoord0);
            float p = float(percent) / 100.0f;
            float alpha = 1.0f;

            if (dir == 0 ) {
                alpha = 1.0 - step(p, qt_TexCoord0.x);
            } else if (dir == 1){
                alpha = 1.0 - step(p, 1.0 - qt_TexCoord0.x);
            } else if (dir == 2) {
                alpha = 1.0f - step(p, qt_TexCoord0.y);
            } else if (dir == 3) {
                alpha = 1.0f - step(p, 1.0 - qt_TexCoord0.y);
            }
            fragColor = vec4(color.rgb, alpha);
        }
```
效果比较简单，以从左向右为例(dir == 0), 说明一下:

先是把percent 归一化处理 (float p = percent / 100.0)，

纹理坐标qt_TexCoord0.x的取值范围为 0 - 1，按照Qml的坐标系统，左边为0，右边为1。

之后纹理坐标与p进行比较，坐标小于p则显示(透明度为1)，大于p则不显示(透明度为0). (也可以直接用discard丢弃片段)

step是glsl内置函数，step(p, qt_TexCoord0.x) 就是x小于p返回0，大于等于p返回1。 结果正好与上面分析的相反，用1 减去即可： alpha = 1.0 - step(p, qt_TexCoord0.x);

最终输出颜色即可:

fragColor = vec4(color.rgb, alpha);
