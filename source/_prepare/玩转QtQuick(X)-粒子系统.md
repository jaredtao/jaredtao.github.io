---
title: 玩转QtQuick(X)-粒子系统
photos: /images/qt_extended_48x48.png
tags:
  - Qt
  - Qml
  - QtQuick
categories: 玩转Qml
abbrlink: 18621
date: 2021-01-21 13:44:23
---
- [简介](#简介)
- [Qt Quick的粒子系统](#qt-quick的粒子系统)

# 简介

这是《玩转QtQuick》系列文章的第X篇，主要是介绍Qt Quick的粒子系统。

因为Qt官方文档写的比较全面，所以本文主要是对官方文档的翻译，同时会补充一些个人理解。

翻译主要参考Qt5.15的文档，适当做了一些调整，尽量信达雅，尽量说人话。

下面翻译开始

# Qt Quick的粒子系统

Qt Quick的粒子系统，包含四种主要的类型：`ParticleSystem`,`Painter`,`Emitters`和`Affectors`。

`ParticleSystem`和其它类型结合在一起使用，管理公用的“时间线”。

`Painter`,`Emitters`和`Affectors`必须拥有相同的`ParticleSystem`才能互相交互。

