---
title: Qt3D探路笔记1
photos: /img/avatar.jpg
tags:
  - Qt
  - Qt实用技能
categories: Qt进阶之路
abbrlink: 55914
date: 2019-08-4 13:22:23
---

## 简介

Qt3D已经推出有一段时间了，除了官方和KDAB的文档，相关资料非常稀缺。

涛哥研究过一段时间Qt3D，这里把自己探路的一些笔记分享出来, 抛砖引玉吧。

我们看图说话。

## 3D场景

首先得创建一个3D场景

![预览](/images/Qt3D/2.png)

安卓模拟器也跑一跑

![预览](/images/Qt3D/2_Android.png)

## 三角形

随便画点三角形，试试牛刀杀鸡。

![预览](/images/Qt3D/3.png)

![预览](/images/Qt3D/3_Android.png)

创建了4个3D场景，放在了一起。

左上角为顶点绘制的三角形，4个点+ TriangleFan的方式绘制。

右上角是 索引+顶点绘制的线框模式两个三角形

左下角为一次绘制两个三角形（顶点数据包含两个三角形）

右下角为绘制彩色的三角形(顶点数据之外，增加色彩数据)

## 纹理

贴纹理

![预览](/images/Qt3D/5.png)

![预览](/images/Qt3D/5_Android.png)

## 立方体

![预览](/images/Qt3D/7.png)

![预览](/images/Qt3D/7_Android.png)

## 动态创建立方体

![预览](/images/Qt3D/8.png)

![预览](/images/Qt3D/8_Android.png)

## 几何图形合影

![预览](/images/Qt3D/16.png)

## 立方体贴图

![预览](/images/Qt3D/10.png)

## 天空盒

![预览](/images/Qt3D/11.png)

![预览](/images/Qt3D/11_Android.png)

## 3D文字

![预览](/images/Qt3D/14.gif)

## 纳米军团

![预览](/images/Qt3D/15.png)

## 代码

都挺简单的，不做说明了，直接看源码吧。

https://github.com/jaredtao/Qt3D-Learn