---
title: Qt实用技能5-信号槽的应用细节
date: 2019-07-23 18:44:23
photos: /img/avatar.jpg
tags: 
    - Qt 
    - QtCreator
    - Qt实用技能
categories: Qt进阶之路
---

- [简介](#%e7%ae%80%e4%bb%8b)
  - [无moc的信号槽实现](#%e6%97%a0moc%e7%9a%84%e4%bf%a1%e5%8f%b7%e6%a7%bd%e5%ae%9e%e7%8e%b0)
- [信号-槽使用细节](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e4%bd%bf%e7%94%a8%e7%bb%86%e8%8a%82)
  - [connect函数的五种写法](#connect%e5%87%bd%e6%95%b0%e7%9a%84%e4%ba%94%e7%a7%8d%e5%86%99%e6%b3%95)
  - [字符串式 与 函数指针式 的差异](#%e5%ad%97%e7%ac%a6%e4%b8%b2%e5%bc%8f-%e4%b8%8e-%e5%87%bd%e6%95%b0%e6%8c%87%e9%92%88%e5%bc%8f-%e7%9a%84%e5%b7%ae%e5%bc%82)
  - [信号-槽的重载 QOverload和qOverload](#%e4%bf%a1%e5%8f%b7-%e6%a7%bd%e7%9a%84%e9%87%8d%e8%bd%bd-qoverload%e5%92%8cqoverload)
- [参考文献](#%e5%8f%82%e8%80%83%e6%96%87%e7%8c%ae)


## 简介

本文是《Qt实用技能》系列文章的第五篇，涛哥在这里讨论Qt信号-槽相关的知识点。

信号-槽是Qt框架中最核心的机制，也是每个Qt开发者必须掌握的技能。


### 无moc的信号槽实现

Verdigris 

https://woboq.com/blog/verdigris-qt-without-moc.html


https://github.com/NoAvailableAlias/nano-signal-slot

## 信号-槽使用细节

### connect函数的五种写法

### 字符串式 与 函数指针式 的差异

### 信号-槽的重载 QOverload和qOverload


## 参考文献

[1] Qt帮助文档, 搜索关键词 Signals & Slots
[2] IBM文档库 https://www.ibm.com/developerworks/cn/linux/guitoolkit/qt/signal-slot/index.html


