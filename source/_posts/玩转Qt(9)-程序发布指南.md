---
title: 玩转Qt(9)-程序发布指南
photos: /img/avatar.jpg
tags:
  - Qt
  - Qt实用技能
  - Qt发布
categories: 玩转Qt
abbrlink: 57085
date: 2019-09-12 18:44:23
---

- [简介](#%e7%ae%80%e4%bb%8b)
- [背景](#%e8%83%8c%e6%99%af)
- [Qt的安装](#qt%e7%9a%84%e5%ae%89%e8%a3%85)
- [Qt的目录结构](#qt%e7%9a%84%e7%9b%ae%e5%bd%95%e7%bb%93%e6%9e%84)
  - [Qt安装路径](#qt%e5%ae%89%e8%a3%85%e8%b7%af%e5%be%84)
  - [Qt核心路径](#qt%e6%a0%b8%e5%bf%83%e8%b7%af%e5%be%84)
- [HelloDeploy](#hellodeploy)
- [Window编译和发布](#window%e7%bc%96%e8%af%91%e5%92%8c%e5%8f%91%e5%b8%83)
  - [Window 编译](#window-%e7%bc%96%e8%af%91)
  - [Window 发布](#window-%e5%8f%91%e5%b8%83)
  - [VS运行时库](#vs%e8%bf%90%e8%a1%8c%e6%97%b6%e5%ba%93)
  - [常见的错误处理](#%e5%b8%b8%e8%a7%81%e7%9a%84%e9%94%99%e8%af%af%e5%a4%84%e7%90%86)
    - [应用程序无法正常启动](#%e5%ba%94%e7%94%a8%e7%a8%8b%e5%ba%8f%e6%97%a0%e6%b3%95%e6%ad%a3%e5%b8%b8%e5%90%af%e5%8a%a8)
    - [启动失败 - no Qt platform plugin](#%e5%90%af%e5%8a%a8%e5%a4%b1%e8%b4%a5---no-qt-platform-plugin)
    - [OpenGL Context 创建失败](#opengl-context-%e5%88%9b%e5%bb%ba%e5%a4%b1%e8%b4%a5)
  - [整理](#%e6%95%b4%e7%90%86)
  - [简单裁剪](#%e7%ae%80%e5%8d%95%e8%a3%81%e5%89%aa)
    - [删减dll](#%e5%88%a0%e5%87%8fdll)
    - [删减plugins](#%e5%88%a0%e5%87%8fplugins)
    - [删减qml](#%e5%88%a0%e5%87%8fqml)

# 简介

这次讨论发布Qt应用程序的知识点。

# 背景

有很多人向涛哥询问，Qt程序发布的相关问题，网络上虽然可以搜到一大堆教程，但是可靠的比较少。

所以这次尽我所能，全面、详细地整理一些Qt程序发布的知识点，希望能帮助到更多人。

对老手来说，很多坑都踩过了，无非就是把正确的dll放在正确的路径。

对新手来说，细节上能多说几句，都将是莫大的帮助，少走弯路，节省几个小时、甚至几天都是有可能的。

如果有疏漏、错误，也欢迎大家补充、指正。

# Qt的安装

Qt官网下载地址在这： http://download.qt.io/official_releases

离线安装包 或者 在线安装包 都行。

关于Qt版本的选择，涛哥建议：

    体验新特性，就用最新版本；项目开发，用长期支持版(LTS)的最后一个修正版本，稳定、bug最少。

可以在Qt官方wiki上查看相关信息 https://wiki.qt.io/Main

目前为止(2019/9/2)，最新版为5.13.0，LTS版本有5.9 和 5.12， 而5.9最后一个修正版本是5.9.8， 5.12则是到5.12.4

![预览](/images/QtDeployment/01.png)

例如上图是5.9.8的离线安装包，提供了windows、mac以及linux三种系统的可执行程序。

其中windows的安装程序"qt-opensource-windoiws-x86-5.9.8.exe", 大小有2.4G，里面

包含了msvc_x86、msvc_x64、mingw、Android等多个版本的Qt工具链。在下载完成，安装

过程中可以分别勾选。其它版本也是类似的。

如何安装Qt，就不细说了，搞不定的去参考入门级教程吧...

# Qt的目录结构

这里假设大家都装好了Qt，先来了解一下Qt的安装路径都有哪些东西。

涛哥用的是Windows 10系统，安装的Qt版本是5.12.4，以此为例来说明，其它系统和版本以实际为准。

## Qt安装路径

涛哥安装在了D:\Qt\Online 路径下, 如图:

![预览](/images/QtDeployment/QtStruct.png)

其中“vcredist”文件夹包含了msvc2015 和 msvc2017的运行时库安装程序(后面会说怎么用,不是msvc编译器不需要)

![预览](/images/QtDeployment/2.png)

"Tools"文件夹，包括QtCreator、OpenSSL库(可选)以及两种版本MinGW(可选)。

![预览](/images/QtDeployment/3.png)

(图中还有Qt3DStudio,可忽略)

“5.12.4”文件夹，是Qt的核心路径, 里面包含多个版本的Qt工具链、头文件、动态链接库等

![预览](/images/QtDeployment/1.png)

这里涛哥安装了msvc2017、msvc2017_64、mingw73_64以及android_x86.

注意msvc2017是x86架构的Qt库，msvc2017_64则是x64架构的。

如果有msvc2013、msvc2015也同理。

## Qt核心路径

接下来看一下重点，Qt的核心路径， 以msvc2017_64文件夹为例

![预览](/images/QtDeployment/4.png)

bin文件夹包含了Qt提供的各种工具exe程序，以及动态链接库的dll

其中工具包括qmake.exe 和 windeployqt.exe，windeployqt.exe是我们今天主要讨论的工具。

动态链接库全部是两份dll，比如Qt5Cored.dll和Qt5Core.dll，文件名末尾带'd'表示debug版本的，另一个不带'd'的是release版本。

![预览](/images/QtDeployment/5.png)

debug版本和release版本的主要区别：debug没有开编译器优化、携带了调试信息，release开了编译器优化O2,去掉了多余的信息

(图中还有pdb文件，是涛哥单独安装的，用来调试Qt源码，可以忽略)

和bin同级的，还有plugins文件夹，包含一些Qt用到的插件

![预览](/images/QtDeployment/6.png)

比如imageformats文件夹中提供了jepg、gif、webp等图片格式的功能支持的插件，platforms文件夹则提供了平台插件，特别是

qwindows.dll这一个，在windows平台是必不可少的。

和bin同级的，另外一个文件夹是'qml'文件夹，包含Qml的各种功能模块。

和bin同级的其它文件夹，resources是WebEngine模块专用的，translations提供了

Qt内置的翻译文件，剩下的和发布无关，就不多说了。

# HelloDeploy

这里新建一个简单的Hello World程序，名字就叫"HelloDeploy"。

同时为了说明问题，涛哥添加一些常用的模块。

在pro文件中，QT += 那一行该写的都写上：

![预览](/images/QtDeployment/8.png)

在main.cpp中包含一下各个模块的头文件，再分别创建一个对象实例，调用一些简单的函数：

![预览](/images/QtDeployment/9.png)

这样一个多模块依赖的程序就写好了。

# Window编译和发布

## Window 编译

这里要特别注意，编译器的选择, 以及编译用的是debug模式还是release模式。

涛哥这里是msvc2017_x64版本

![预览](/images/QtDeployment/10.png)

一般发布用release模式。

![预览](/images/QtDeployment/11.png)

编译完成后，默认在build-xxxx-release/release/文件夹中会生成我们的exe程序。

![预览](/images/QtDeployment/12.png)

![预览](/images/QtDeployment/13.png)

我们将这个exe复制出来，新建一个release文件夹，放进去

![预览](/images/QtDeployment/14.png)

这时候可以尝试双击运行它，会提示缺少dll

![预览](/images/QtDeployment/15.png)

## Window 发布

发布程序，其实就是把exe程序依赖的dll和相关资源都放在一起，保证双击运行即可。

我们前面提过的windeployqt.exe，是Qt提供的命令行工具，能帮助我们自动把需要的dll或资源复制过来。

1. 我们先打开一个命令行
   
可以从开始菜单找到Qt提供的命令行

![预览](/images/QtDeployment/16.png)

注意选对版本。这种命令行在启动时已经设置好了QT的环境变量，可以直接输入windeployqt.exe

也可以用普通的命令行，使用windeployqt.exe时带上绝对路径即可。
    
涛哥一般用普通的命令行，因为绝对路径不易出错。

2. cd到release目录
   
这里说一个windows启动命令行的小技巧：在release文件夹中，按住键盘shift键，然后按鼠标右键，弹出的右键菜单，
    
会比普通的右键菜单多一个“在此处打开命令窗口”，点击就能在release文件夹打开命令行窗口。

![预览](/images/QtDeployment/17.png)

如果没有这个功能，就得手动输入cd指令，进入release路径。

![预览](/images/QtDeployment/18.png)

3. 执行windeployqt命令

这里通过绝对路径来使用windeployqt：

d:\qt\Online\5.12.4\msvc2017_64\bin\windeployqt.exe HelloDeploy.exe

![预览](/images/QtDeployment/19.png)

HelloDeploy这个程序还用到了Qml，用到Qml的程序，要给windeployqt加上qmldir参数，写上你的Qml文件所在文件夹

(没用到qml的程序，不要加这一步)

d:\qt\Online\5.12.4\msvc2017_64\bin\windeployqt.exe HelloDeploy.exe --qmldir .\qml

![预览](/images/QtDeployment/20.png)

写好windeployqt命令后按回车执行

![预览](/images/QtDeployment/21.png)

正确执行后，release文件夹下，多了很多dll，以及一些文件夹。

![预览](/images/QtDeployment/22.png)

这时候我们双击运行HelloDeploy.exe, 就可以正常启动了。

将整个文件夹压缩或拷贝到其它没有Qt环境的电脑上，也是可以启动的。

只要dll齐备了，制作安装包也不是问题。(后续有时间，我再写安装包制作的教程)

## VS运行时库

如果是VS编译的程序，需要将QT路径下对应的vcredist_xxx.exe带上。

如果其它电脑上有vs运行时则可以直接运行，如果没有，就需要运行一下vs运行时安装包。

或者将运行时库里面的dll复制出来即可。

一般在VS的安装路径，都有展开的dll，可以直接拷贝。

例如，涛哥电脑上的vs2017路径如下：

![预览](/images/QtDeployment/msvc.png)

按实际的路径找到这几个dll，全部拷贝即可。注意x86和x64，别拿错了。


## 常见的错误处理

一般使用windeployqt，大部分库都能自动拷贝，不需要手动处理，

只有极少数情况下，windeployqt跑完，会缺失一些库，还要手动处理一下。

遇到这种情况，用依赖检查工具Dependencies即可快速定位问题。

Dependencies下载链接:  https://github.com/lucasg/Dependencies

Dependencies 下载好，点击"DependenciesGui.exe"就可以打开界面。注意是名字带Gui的那个，不带gui的“Dependencies.exe”是命令行程序。

![预览](/images/QtDeployment/depency.jpg)

下面列举一些常见的错误信息

### 应用程序无法正常启动

![预览](/images/QtDeployment/23.png)

最容易出现这种错误的情况是，程序是64位编译出来的，而同级目录下的dll是32位的，

或者同级目录下没有dll，但是环境变量中指向了32位的dll。(所以涛哥没有设置环境变量)

32位和64位倒过来也是。

如果dll版本是匹配的，还有可能出现的情况是缺少第三方库。

这里说一个检查依赖的方法：

将HelloDeploy.exe重命名为HelloDeploy.dll，然后用Dependencies打开，就可以查看少哪些库

![预览](/images/QtDeployment/24.png)

如上图，红色问号的表示缺少的库。

找齐了依赖的库，再把程序的扩展名改回exe即可。

### 启动失败 - no Qt platform plugin

![预览](/images/QtDeployment/27.png)

这种情况，是QT路径下的 plugins/platforms/qwindows.dll文件没有复制过来。

注意这个dll文件直接复制到exe同级是不起作用的，要放在exe程序同级的platforms文件夹里，或者同级

的plugins/platforms文件夹里

### OpenGL Context 创建失败

![预览](/images/QtDeployment/25.png)

这种情况，一般是OpenGL相关的库没有复制过来，补上就好了

![预览](/images/QtDeployment/26.png)

## 整理

我们看到，exe同级目录下，windeployqt将一堆的文件夹放在了那里，有些混乱。

![预览](/images/QtDeployment/28.png)

涛哥观察并验证了一下，其实可以做个简单的整理。

![预览](/images/QtDeployment/29.png)

Qt开头的文件夹都是qml的模块，剩下的文件夹除了translations都是Qt的插件，

所以新建两个文件夹qml和plugins, 分别把qml模块和插件归入其中。

![预览](/images/QtDeployment/30.png)

这样的结构，和QT安装路径下的结构是相似的。

这也正是Qt支持的插件加载路径、qml模块加载路径。

同级的dll则是windows系统默认的动态库加载规则，不方便修改

可以参考msdn：

https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order

## 简单裁剪

如果你熟悉Qt的各个模块，可以进行一些裁剪。以下都是些个人经验。

不熟悉请慎重！

不熟悉请慎重！

不熟悉请慎重！

(当然静态编译也是一种裁剪的途径)

### 删减dll

首先可以把单元测试的dll去掉

Qt5Test.dll

Qt5QuickTest.dll

如果没用到windows扩展，Qt5WinExtras.dll也可以去掉

其次，如果你不需要内置的翻译文件，translations文件夹也可以删掉

![预览](/images/QtDeployment/32.png)

### 删减plugins

再来看一下plugins：

其中platforms是必不可少的，剩下的HelloDeploy都没用到，可以去掉。

![预览](/images/QtDeployment/33.png)

常见程序会用的包括:
 
 imageformats 图片格式支持
 
 iconengines 小图标功能

 sqldrivers 数据库驱动，这个保留用到的数据库足够了

 其他的看情况删减。

### 删减qml

最后看一下Qml文件夹，如果程序完全没用qml，直接删掉就好了。

![预览](/images/QtDeployment/34.png)

按windeployqt给HelloDeploy提供的这些，逐个文件夹来说:

* Qt/labs 一般不推荐Qml中引入labs中的实验品，但是有些情况下功能缺失，只能引入。

如果Qml中使用了Quick.Dialog(不是labs.Dialog)，它本身还是依赖的labs中的东西，一般是folderlistmodel和settings，

这时候还是不要动labs了，就按照windeployqt给的放着。
    
* Qt/WebSockets Qml的Websocket功能，用了就放着，没用可以删掉。

* QtGraphicalEffects Qml的一些ShaderEffect特效，用了就放着，没用到可以删掉

* QtMultimedia Qml的多媒体模块，用了就放着，没用到可以删掉

* QtQml/Models.2 数据Model, 经常用。

* QtQuick 这里面大部分都是Qml中常用的，QtQuick/Extras可以按情况删掉

* QtQuick.2 常用的

* QtTest 单元测试，删掉吧

* QtWinExtras Windows扩展，没用到可以去掉