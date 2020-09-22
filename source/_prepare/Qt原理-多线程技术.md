---
title: Qt实用技能2-用好qmake
date: 2019-05-29 18:44:23
photos: /img/avatar.jpg
tags: 
    - Qt 
    - QtCreator
    - Qt实用技能
categories: Qt进阶之路
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [qmake简介](#qmake%E7%AE%80%E4%BB%8B)
- [添加第三方库](#%E6%B7%BB%E5%8A%A0%E7%AC%AC%E4%B8%89%E6%96%B9%E5%BA%93)
  - [示例1 - 直接链接库的全路径](#%E7%A4%BA%E4%BE%8B1---%E7%9B%B4%E6%8E%A5%E9%93%BE%E6%8E%A5%E5%BA%93%E7%9A%84%E5%85%A8%E8%B7%AF%E5%BE%84)
  - [示例2 - 路径中包含空格等特殊字符，用引号括起来。](#%E7%A4%BA%E4%BE%8B2---%E8%B7%AF%E5%BE%84%E4%B8%AD%E5%8C%85%E5%90%AB%E7%A9%BA%E6%A0%BC%E7%AD%89%E7%89%B9%E6%AE%8A%E5%AD%97%E7%AC%A6%E7%94%A8%E5%BC%95%E5%8F%B7%E6%8B%AC%E8%B5%B7%E6%9D%A5)
  - [示例3 - 分别指定路径和库](#%E7%A4%BA%E4%BE%8B3---%E5%88%86%E5%88%AB%E6%8C%87%E5%AE%9A%E8%B7%AF%E5%BE%84%E5%92%8C%E5%BA%93)
  - [示例4 - 分平台条件链接](#%E7%A4%BA%E4%BE%8B4---%E5%88%86%E5%B9%B3%E5%8F%B0%E6%9D%A1%E4%BB%B6%E9%93%BE%E6%8E%A5)
  - [原理](#%E5%8E%9F%E7%90%86)
- [影子构建](#%E5%BD%B1%E5%AD%90%E6%9E%84%E5%BB%BA)
  - [指定目标路径](#%E6%8C%87%E5%AE%9A%E7%9B%AE%E6%A0%87%E8%B7%AF%E5%BE%84)
  - [指定中间件生成路径](#%E6%8C%87%E5%AE%9A%E4%B8%AD%E9%97%B4%E4%BB%B6%E7%94%9F%E6%88%90%E8%B7%AF%E5%BE%84)
- [拷贝资源](#%E6%8B%B7%E8%B4%9D%E8%B5%84%E6%BA%90)
  - [拷贝资源示例](#%E6%8B%B7%E8%B4%9D%E8%B5%84%E6%BA%90%E7%A4%BA%E4%BE%8B)
  - [编译前拷贝](#%E7%BC%96%E8%AF%91%E5%89%8D%E6%8B%B7%E8%B4%9D)
- [安装](#%E5%AE%89%E8%A3%85)
- [结束语](#%E7%BB%93%E6%9D%9F%E8%AF%AD)


## 简介

本文是《Qt实用技能》系列文章的第二篇，涛哥将教大家，一些qmake的实用技巧。部分地方也会说一下原理，让大家知其然，知其所以然。

工欲善其事，必先利其器。

这个系列，全是干货！

注：文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## qmake简介

qmake是Qt的构建工具，主要作用是解析pro格式的项目文件、生成编译规则(Makefiles或其它)。

qmake是一个比较古老的工具，很多功能使用perl脚本实现，涛哥在其它地方就没怎么见过使用perl脚本的代码/项目。

Qt官方之前开发的Qbs，后来又宣布不再更新，现在又大力支持CMake。。。

在这样的背景下，qmake依然是当下主要的构建工具，所以qmake的一些技巧还是有必要掌握的。

qmake本身作为一个可执行程序，也是有一些参数的，但这不是本文的重点，本文的重点都在pro文件里。

pro文件中，除了常规的组织项目结构外，还可以做很多事情, 比如 指定编译选项、链接选项、制定目标生成规则、扩展编译规则 等等。

pro文件中的qmake语法，包括 变量声明和使用、内建变量、替换函数、测试函数等，帮助文档都有详细的介绍。

搜索关键词为qmake， 或者和普通的类查看帮助文档方式一样，光标放在pro文件要查看的变量上，按F1就能看到相应的说明。

![预览](/images/Qt2/3.gif)

涛哥就不赘述了，后面用到的会单独说明。

## 添加第三方库

c++开发，使用第三方库也是家常便饭了，这是一个必备的技能。

这里首选的方法，是使用QtCreator提供的添加库UI。在pro文件里(或者项目文件夹), 鼠标右键->添加库，然后根据自己的需要下一步、下一步点一下即可。

![预览](/images/Qt2/1.gif)

熟练的人也可以直接按pro语法(perl语法)写，给LIBS变量赋值。

下面给几个示例，至于动态库/静态库的差异，大家自己实践吧。

### 示例1 - 直接链接库的全路径
```shell
  LIBS += c:/mylibs/math.lib
```
我们都知道windows系统默认的路径分割符是'\'，但在qmake中要写成'\\'才行。qmake也支持写成'/'，其它unix系统又都是'/'，

所以干脆都写成'/'，方便处理。

### 示例2 - 路径中包含空格等特殊字符，用引号括起来。
```shell
  LIBS += "C:/mylibs/extra libs/extra.lib"
```

### 示例3 - 分别指定路径和库
```shell
LIBS += "C:/mylibs/extra libs" -lextra
```
这里的LIBS指定要链接的库，'-L'是指定链接库的路径，'-l'指定要链接的库名称

名称可以省略lib前缀和 扩展名后缀，Qt会自动处理。 后缀包括 '.so' '.dll' '.dylib' 等。

### 示例4 - 分平台条件链接
```shell
  win32:LIBS += "C:/mylibs/extra libs/extra.lib"
  unix:LIBS += "-L/home/user/extra libs" -lextra
```
条件链接可以很方便地实现不同平台链接不同的库。

这里的 win32 unix 是在选择了不同的编译器环境时，qmake分别预置的变量。

(比如win32平台相关的变量，可以参考msvc的配置文件:  [QTDIR]/mkspecs/win32-msvc/qmake.conf 和 [QTDIR]/mkspecs/common/msvc-desktop.conf)

### 原理

Qt内置了一些perl脚本，在执行qmake解析时会包含这些脚本。其中一些脚本会来处理这个LIBS变量，将其转换成编译器/链接器的参数。

内置的脚本路径在[QTDIR]/mkspecs/features文件夹下，扩展名为prf。

![预览](/images/Qt2/2.png)

后续的很多变量，也是一样的原理, 只是处理方式各不相同。

很多pro文件的语法、功能实现，都可以参考这些prf来实现。

(注意：不熟悉的同学，不要乱改prf，容易改坏)

Qt程序员都知道的一件事：有时候修改了信号/槽相关的代码，不能正常运行，要重新qmake一下，才会生效。

本质上就是在重新触发[QTDIR]/mkspecs/features/moc.prf这个脚本。

(多少年了，都没有修好Moc生成问题，可见qmake的古老...)

## 影子构建

影子构建，就是编译生成的产物和源代码在不同的文件夹。这样可以防止源代码文件夹被污染。

QtCreator默认导入pro工程时，就会生成一个影子构建路径。比如这样：

![预览](/images/Qt2/4.png)

```
F:\Dev\Qt\build-HelloTaoQuick-Desktop_Qt_5_12_3_MSVC2017_64bit-Debug
```
之后编译项目时生成的中间文件及目标文件，都在这个文件夹中。

这个路径很长，而且编译器或者编译选项不同时都有可能不一样。

有时候要做一些特定的操作 比如目标exe生成到特定目录、拷贝资源文件等等，直接用这个路径会不太方便/不太可靠，我们需要一些定制。

### 指定目标路径

```shell
DESTDIR = $$PWD/bin
```
通过给DESTDIR变量赋值， 可以指定生成的lib/exe放在哪个目录下

'PWD'是qmake内置变量，'$$'是内置变量取值的写法。'/bin'是字符串拼接在变量后面。

更多内置变量可以参考qmake帮助文档，以及这篇文档[隐藏的qmake文档](https://wiki.qt.io/Undocumented_QMake)

当然也可以参考那一堆prf和conf文件。

### 指定中间件生成路径

可以通过这几个变量指定中间件生成的路径

```qmake
config(debug, debug|release) {
    OBJECTS_DIR = build/debug/obj
    MOC_DIR = build/debug/moc
    RCC_DIR = build/debug/rcc
    UI_DIR = build/debug/uic
} else {
    OBJECTS_DIR = build/release/obj
    MOC_DIR = build/release/moc
    RCC_DIR = build/release/rcc
    UI_DIR = build/release/uic
}

```

config(debug, debug|release) 是一个条件表达式，可以理解为 

```qmake
if (debug === true) {

} else if (release == true) {

}
```
注意: 按照perl语法，那个左大括号'{'不能换行,要和前面的表达式在同一行。(有人自作聪明换行，被坑了呢😄)

上面这种指定中间件路径的方式，在QtCreator中有默认路径所以没有太大意义，不过在命令行编译时这样写却很有用。

## 拷贝资源

pro可以实现，在编译代码时，拷贝一些文件到指定的路径下

### 拷贝资源示例

这里以TaoQuick为例,来说明：

我在TaoQuick库目录下，有个叫qmldir的文件，需要在编译代码时自动拷贝到bin目录下。(先别管这个文件干嘛的，下一篇文章会说)

![预览](/images/Qt2/5.png)

关键目录结构如下：
```
TaoQuick
    TaoQuick.pro
    - bin
        -TaoQuick
    - TaoQuickCore
        TaoQuickCore.pro
        - Qml
            qmldir
```
那么我在TaoQuickCore.pro文件中的写法如下：

```shell
!equals(_PRO_FILE_PWD_, $$DESTDIR) {
    copy_qmldir.target = $$DESTDIR/qmldir
    copy_qmldir.depends = $$_PRO_FILE_PWD_/qmldir
    win32 {
        copy_qmldir.target ~= s,/,\\\\,g
        copy_qmldir.depends ~= s,/,\\\\,g
    }
    copy_qmldir.commands = $${QMAKE_COPY_FILE} $${copy_qmldir.depends} $${copy_qmldir.target}
    QMAKE_EXTRA_TARGETS += copy_qmldir
}

```
‘!equals(_PRO_FILE_PWD_, $$DESTDIR)’ 这一句是执行条件，即: 目标路径不等于pro文件所在路径时 执行下面的操作。

剩下的事情就是在创建一个"编译目标"（Target），将这个编译目标添加到QMAKE_EXTRA_TARGETS变量中就行了。

熟悉MakeFiles的同学应该都清楚什么是"目标"。不懂MakeFiles也没关系，这里的目标就理解为自己声明的一个变量即可。

这个变量有三个很重要的"子变量":

copy_qmldir.target 指定目标文件所在的路径   (这里理解成要拷贝到哪去)
copy_qmldir.depends 指定依赖文件所在的路径  (这里理解成从哪里拷贝)
copy_qmldir.commands 指定拷贝操作的执行命令 (就是怎么拷贝)

QMAKE_COPY_FILE 这个变量来自前面说过的[隐藏的qmake文档](https://wiki.qt.io/Undocumented_QMake)

qmake会在解析pro文件时，自动替换成平台相应的拷贝命令。 windows 平台就是 copy /y

注意windows的copy指令，路径分隔符得写成 '\\'才行。所以有了下面的特殊处理：

```
    win32 {
        copy_qmldir.target ~= s,/,\\\\,g
        copy_qmldir.depends ~= s,/,\\\\,g
    }
```
‘s,/,\\\\,g’ 是一个正则表达式，作用是把‘/’替换成‘\\’ 。s表示开头，g表示结尾。

VAR~= REGEXP 是对变量VAR执行REGEXP这个正则表达式

### 编译前拷贝

如果想在编译之前，先把资源拷贝完成，只需要前面的基础上，添加一句

```
    PRE_TARGETDEPS += $$copy_qmldir.target
```
也就是把"目标"加到 PRE_TARGETDEPS变量

```shell
!equals(_PRO_FILE_PWD_, $$DESTDIR) {
    copy_qmldir.target = $$DESTDIR/qmldir
    copy_qmldir.depends = $$_PRO_FILE_PWD_/qmldir
    win32 {
        copy_qmldir.target ~= s,/,\\\\,g
        copy_qmldir.depends ~= s,/,\\\\,g
    }
    copy_qmldir.commands = $${QMAKE_COPY_FILE} $${copy_qmldir.depends} $${copy_qmldir.target}
    QMAKE_EXTRA_TARGETS += copy_qmldir
    PRE_TARGETDEPS += $$copy_qmldir.target
}

```

## 安装

pro中还有一种INSTALL功能，可以执行文件拷贝。

和编译期拷贝 类似，INSTALL用起来更简单无脑一些，而且INSTALL只在执行make install指令时，才会拷贝资源。

还是以TaoQuick为例, 我有一堆文件，需要在make install时，安装到Qt的Qml路径中

![预览](/images/Qt2/6.png)

如上图所示所有的文件, 除了TaoQuickDesigner.pri, 都要按照这个结构拷贝。

(这个pri文件是pro文件的一小部分，可以直接在pro中通过include引入。

pri和pro语法一样，但是qmake不直接识别pri，只识别pro

pri一般用来写一些公用的部分，让多个pro公用)

拷贝整个文件夹是一种做法, 当然为了精确地控制要拷贝的内容，可以写成下面这样：

```
taoquick_designer.files = $$PWD/designer/TaoQuick.metainfo
taoquick_designer.path = $$[QT_INSTALL_QML]/$${uri}/designer

toaquick_qmldir.files = $$PWD/qmldir
toaquick_qmldir.path = $$[QT_INSTALL_QML]/$${uri}

taoquick_qml_buttons.files = $$PWD/BasicComponent/Button/*.qml
taoquick_qml_buttons.path = $$[QT_INSTALL_QML]/$${uri}/BasicComponent/Button

taoquick_qml_mouse.files = $$PWD/BasicComponent/Mouse/*.qml
taoquick_qml_mouse.path = $$[QT_INSTALL_QML]/$${uri}/BasicComponent/Mouse

taoquick_qml_others.files = $$PWD/BasicComponent/Others/*.qml
taoquick_qml_others.path = $$[QT_INSTALL_QML]/$${uri}/BasicComponent/Others

taoquick_qml_progress.files = $$PWD/BasicComponent/Progress/*.qml
taoquick_qml_progress.path = $$[QT_INSTALL_QML]/$${uri}/BasicComponent/Progress

taoquick_degisner_images.files = $$PWD/designer/images/*.png
taoquick_degisner_images.path = $$[QT_INSTALL_QML]/$${uri}/designer/images

INSTALLS  += taoquick_designer toaquick_qmldir taoquick_qml_buttons taoquick_qml_mouse taoquick_qml_others taoquick_qml_progress taoquick_degisner_images

```
自定义一个变量，然后其子变量files指定要拷贝的文件，子变量path指定目标路径。

把自定义变量加入INSTALLS变量就行了。

QT_INSTALL_QML也是一个内置变量，默认值为[QTDIR]/qml。

之后只要执行以下命令，就能完成资源拷贝。
```shell
qmake
make 
make install
```

当然QtCreator中也能执行make install

如下图所示:

![预览](/images/Qt2/QtCreator-install.png)

任意编译器kit都可以，项目->构建步骤->添加build步骤->Make，添加之后在make参数中输入install。最后重新构建工程，即可完成安装。

## 结束语

以上案例，大部分都在TaoQuick项目中实践过，可以去Github参考。

[TaoQuick](https://github.com/jaredtao/TaoQuick)