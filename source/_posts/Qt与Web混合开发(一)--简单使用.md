---
title: Qt与Web混合开发(一)--简单使用
photos: /img/avatar.jpg
tags:
  - Qt
  - Qt实用技能
categories: Qt进阶之路
abbrlink: 39038
date: 2020-03-04 19:44:23
---
- [前言](#前言)
- [简介](#简介)
- [Qt的Web方案](#qt的web方案)
  - [Quick WebGL Stream](#quick-webgl-stream)
  - [Qt WebAssembly](#qt-webassembly)
  - [Qt WebEngine/WebView](#qt-webenginewebview)
  - [QtWebEngine的更新情况](#qtwebengine的更新情况)
  - [WebEngine的架构](#webengine的架构)
  - [WebEngine的平台要求](#webengine的平台要求)
    - [Windows](#windows)
    - [MacOS](#macos)
    - [Linux](#linux)
  - [WebView](#webview)
  - [WebEngine的使用](#webengine的使用)
    - [WebEngine Widget最简Demo](#webengine-widget最简demo)
      - [源代码](#源代码)
      - [运行结果](#运行结果)
      - [最小发布包](#最小发布包)
    - [WebEngine Qml最简Demo](#webengine-qml最简demo)
      - [源码](#源码)
      - [运行结果](#运行结果-1)
      - [最小发布包](#最小发布包-1)

# 前言

《Qt与Web混合开发》系列文章，主要讨论Qt与Web混合开发相关技术。

这类技术存在适用场景，例如：Qt项目使用Web大量现成的组件/方案做功能扩展，

Qt项目中性能无关/频繁更新迭代的页面用html单独实现，Qt项目提供Web形式的SDK给

用户做二次开发等等，或者是Web开发人员齐全而Qt/C++人手不足，此类非技术问题，

都可以使用Qt + Web混合开发。

(不适用的请忽略本文)

# 简介

第一篇文章，会先整体介绍一下Qt的各种Web方案,再提供简单的Demo，并做一些简要的说明。


# Qt的Web方案

Qt提供的Web方案主要包括 WebEngine/WebView、Quick WebGL Stream、QtWebAssembly三种。

## Quick WebGL Stream

可以参考Qt官方的WebGL Stream介绍文档

https://resources.qt.io/cn/qt-quick-webgl-release-512
​
resources.qt.io
WebGL Stream在5.12中正式发布，其本质是一种通信技术，将已有的QtQuick程序中渲染指令和数据，通过socket传输给Web端，由WebGL实现界面渲染。

其使用方式非常的简单，无需修改源码，应用程序启动时，带上端口参数，例如：

./your-qt-application -platform webgl:port=8998
(相当于应用程序变成了一个服务器端程序)

这样程序就在后端运行，看不到界面了，之后浏览器打开本地网址 localhost:8998 或者内网地址/映射后的公网地址，就能在浏览器中看到程序页面。

WebGL Stream的应用不多，Qt官方给的案例是：欧洲某工厂的大量传感器监测设备，都以WebGL Stream的方式运行Qt 程序，本身都不带显卡和显示器，而在控制中心的显卡/显示器上，通过Web打开网页的方式，查看每个设备的运行状况。因此节约了大量显卡/显示器的成本。类比于网吧的无硬盘系统。

涛哥相信，未来结合5G技术会有不错的应用场景。

## Qt WebAssembly

Qt WebAssembly技术，在5.13中正式发布。本质是把Qt程序编译成浏览器支持的二进制文件，由浏览器加载运行。

一方面可以将现有的Qt程序编译成Web，另一方面可以用Qt/C++来弥补Web程序的性能短板。

Qt WebAssembly在使用细节上还有一些坑的地方，需要踩一踩。后续我再写文章吧。

## Qt WebEngine/WebView

Qt提供了WebEngine模块以支持Web功能。

Qt WebEngine基于google的开源浏览器chromium实现，类似的项目还有[cef](https://github.com/chromiumembedded/cef)、[miniblink](https://github.com/weolar/miniblink49)等等。

QtWebEngine可以看作是一个完整的chromium浏览器。

（WebView是同类的方案，稍微有些区别。后文再说。）

## QtWebEngine的更新情况

浏览器技术十分的庞大，这里先不深入展开，先来关注一下Qt WebEngine对chromium的跟进情况。

数据来源于[Qt wiki](https://wiki.qt.io/)，Qt每个版本的change log

|Qt版本|chromium后端|chromium安全更新|
|--|--|--|
|5.9.0|56||
|5.9.1|-|59.0.3071.104|
|5.9.2|-|61.0.3163.79|
|5.9.3|-|62.0.3202.89|
|5.9.4|-|63.0.3239.132|
|5.9.5|-|65.0.3325.146|
|5.9.6|-|66.0.3359.170|
|5.9.7|-|69.0.3497.113|
|5.9.8|-|72.0.3626.121|
|5.9.9|-|78.0.3904.108|
|5.12.0|69||
|5.12.1||71.0.3578.94|
|5.12.2||72.0.3626.121|
|5.12.3||73.0.3683.75|
|5.12.4||74.0.3729.157|
|5.12.5||76.0.3809.87|
|5.12.6||77.0.3865.120|
|5.12.7||79.0.3945.130|
|5.14.0|77||
|5.14.1||79.0.3945.117|

可以看到Qt在WebEngine模块，一直持续跟进Chromium的更新。

当前(2020/3/4)最新的chromium版本是80。

## WebEngine的架构

QtWebEngine提供了C++和Qml的接口，可以在Widget/Qml中渲染HTML、XHTML、SVG，

也支持CSS样式表和JavaScript脚本。

QtWebEngine的架构图如下

![](/images/Web/1.png)

基于Chromium封装了一个WebEngineCore模块，在此之上，

WebEngine Widgets模块专门用于Widget项目，

WebEngine 模块用于Qml项目，

WebEngineProcess则是一个单独的进程，用来渲染页面、运行js脚本。

Web在单独的进程里，我们开发的时候知道这一点就好了，不需要额外关注，

只要在发布的时候，带上QTDIR目录下的可执行程序QtWebEngineProcess即可。

(这里提一下底层实现原理，使用了进程间共享OpenGL上下文的方式, 实现多个进程的UI混合在一起)

## WebEngine的平台要求

(以Qt5.12为参考)

首先一条是：不支持静态编译 (因为依赖的chromium、chromium本身的依赖库 不能静态编译)

接下来再看看各平台的要求和限制：

### Windows

编译器要 Visual Studio 2017 version 15.8 以上

系统环境要 Windows 10 SDK

默认只支持X64版本，如果要x86版本，要自己编译qt源码。

### MacOS

* MacOS 10.12以上
  
* XCode 8.3.3以上
  
* MacOS 10.12以上 SDK

不支持32-bit

不兼容 Mac App Store (chromium使用了私有api，App Sandbox和chromium Sandbox优先级问题)

### Linux

编译器要 clang， 或者 gcc 5以上

需要pkg-config来探测依赖库，dbus-1和 fontconfig是必须的。

如果配置了xcb，还要额外配置相关库。

## WebView

Qt还提供了一个WebView组件，可以用来将Web内容嵌入到Qml程序中。(这个没有提供Widget的接口)

WebView组件的实现，使用了平台原生api，在移动端意义重大，特别是在ios平台，使用

原生的web view，这样就能兼容App Store了。

在Windows/MacOS/Linux平台，则是渲染部分还是使用了WebEngine。

WebView的使用可以参考官方例子Minibrowser

![](/images/Web/6.png)

## WebEngine的使用

### WebEngine Widget最简Demo

#### 源代码

这里示例一个最简单的demo, 使用WebEngine Widget模块提供的QWebEngineView：

```c++
//Demo.pro
QT += core gui webenginewidgets

CONFIG += c++11

SOURCES += \
        main.cpp

```
注意pro文件中包含的Qt模块

```C++
//main.cpp
#include <QApplication>
#include <QWebEngineView>
int main(int argc, char **argv)
{
    QApplication app(argc, argv);

    QWebEngineView view;
    view.load(QUrl("https://www.zhihu.com/"));
    view.show();

    return app.exec();
}
```
#### 运行结果

上面代码以打开知乎首页为例，运行结果如下 

![](/images/Web/2.png)

#### 最小发布包

涛哥尝试了在Windows平台，做出可用的最小发布包:

![](/images/Web/3.png)

尺寸在170M左右。这些依赖项中，除了常见的Qt必备项platforms、Qt5Core、Qt5Gui等，

Qt5WebEngineCore是最大的一个，有70M。QtWebEngineProcess.exe是新增加的一个exe程序，

前文说架构图时提到的单独进程就是这个程序实现。

resources/icudtl.dat在其它浏览器引擎中也常看到。

translations/qtwebengine_locales是WebEngine的翻译项，不带可能会发生翻译问题。

Qt5Positioning、Qt5PrintSupport一般不怎么用，但是不带这两个程序起不来。

同时发现Qml和Quick模块也是必须的，Qt5QuickWidgets也用上了。

涛哥查看源码后发现WebEngineCore模块依赖Quick和Qml模块。

### WebEngine Qml最简Demo

再做一个纯Qml的Demo

#### 源码

pro中增加webengine模块即可

```C++
//WebQml.pro
QT += core gui quick qml webengine

CONFIG += c++11

SOURCES += \
        main.cpp

RESOURCES += \
    Qrc.qrc

```
注意初始化。

```C++
//main.cpp
#include <QGuiApplication>
#include <QQuickView>
#include <QtWebEngine>
int main(int argc, char *argv[])
{
    QCoreApplication::setAttribute(Qt::AA_EnableHighDpiScaling);

    QGuiApplication a(argc, argv);
    //初始化。时机在QApp之后，Window/View构造之前。
    QtWebEngine::initialize();

    QQuickView view;
    view.setSource(QUrl("qrc:/main.qml"));
    view.show();
    return a.exec();
}

```
qml导入模块，填入url
```qml
//main.qml
import QtQuick 2.0
import QtWebEngine 1.8
Item {
    width: 800
    height: 600
    WebEngineView {
        anchors.fill: parent
        url: "https://www.zhihu.com"
    }
}

```
#### 运行结果

运行结果和上一个Demo一样

![](/images/Web/4.png)

#### 最小发布包

这回可以去掉Widget模块

![](/images/Web/5.png)

同时也去掉不必要的翻译文件。包大小160M左右，和前面的差别不大。

