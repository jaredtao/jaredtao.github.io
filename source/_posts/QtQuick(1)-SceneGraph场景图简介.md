---
title: QtQuick(1)-SceneGraph场景图简介
tags:
  - Qt
  - Qml
  - QtQuick
categories: 玩转Qml
abbrlink: 18621
date: 2021-01-20 13:44:23
---
- [简介](#简介)
- [Qt Quick 中的“场景图”](#qt-quick-中的场景图)
- [Qt Quick “场景图”的结构](#qt-quick-场景图的结构)
  - [节点](#节点)
  - [预处理](#预处理)
  - [节点所有权](#节点所有权)
  - [材质](#材质)
  - [便捷的节点](#便捷的节点)
- [“场景图”和渲染](#场景图和渲染)
  - [线程渲染循环](#线程渲染循环)
  - [非线程渲染循环 (基本渲染循环和窗口渲染循环)](#非线程渲染循环-基本渲染循环和窗口渲染循环)
  - [使用`QQuickRenderControl`自定义渲染控制](#使用qquickrendercontrol自定义渲染控制)
  - [“场景图”和原生图形API的混合使用](#场景图和原生图形api的混合使用)

## 简介

这是QtQuick 系列文章的第一篇，主要是介绍Qt Quick SceneGraph “场景图”的关键特性、主要架构及实现原理等等。

因为Qt官方文档写的比较全面，所以本文主要是对官方文档的翻译，后面会补充一些个人理解和应用技巧。

翻译主要参考Qt5.15的文档，适当做了一些调整，尽量信达雅，尽量说人话。

下面翻译开始

## Qt Quick 中的“场景图”

Qt Quick 2 使用了专用的“场景图”，然后遍历并通过图形API(例如OpenGL、OpenGL ES、Vulkan、Metal 或Direct 3D)渲染该“场景图”。

将“场景图”用于图形而不是传统的命令式绘图系统(QPainter之类的)，意味着可以在帧之间保留要渲染的场景，并且在渲染开始之前就知道要

渲染的完整图元集。这为许多优化打开了大门，例如：通过批量渲染最大程度减少状态变化、丢弃被遮挡的图元。



再举个具体的例子，假设用户界面包含一个列表，列表有10个节点，其中每个节点都有背景色、图标和文本。

使用传统的绘图技术，这将导致30次绘图调用和30次状态更改。

而“场景图”可以重组原始图元进行渲染，以便在一次调用中绘制所有背景，然后绘制所有图标，最后绘制所有文本，

从而将绘制调用的总数减少到3次。这样可以显著提高硬件的性能。



“场景图”与Qt Quick 2.0 紧密相关，不能单独使用。“场景图”由`QQuickWindow`类管理和渲染，自定义Item类型

可以通过调用`QQuickItem::updatePaintNode()`将其图元添加到“场景图”中。

“场景图”是Item场景的图形表示，它是一个独立的结构，其中包含足以渲染所有节点的信息。

设置完成后，就可以独立于项目状态对其进行操作和渲染。

在许多平台上，“场景图”会在GUI线程准备下一帧状态时，已经在专用渲染线程上进行渲染。

注意：本文列出的许多信息特定于 Qt “场景图”的内置默认行为。如果使用替代的方案时，并非所有概念都适用。

## Qt Quick “场景图”的结构

“场景图” 由许多预定义的节点类型组成，每种类型都有专门的用途。

尽管我们将其称为“场景图”，但更精确的定义是“节点树”。

该树根据Qml场景中的`QQuickItem`类型构建，然后在内部对该场景进行渲染，最终呈现该场景。

“节点” 本身不包含任何 绘制 或者 paint() 虚函数。

“节点树”主要由内建的预定义类型组成，用户也可以添加具有自定义内容的完整子树，包括表示3D模型的子树。

### 节点

对用户而言,最重要的节点是`QSGGeometryNode`。它用来实现自定义图形中的几何形状和材质。

使用`QSGGeometry`可以定义几何坐标，并描述形状或者图元网格。它可以是直线，矩形，多边形，许多

不连续的矩形或者复杂的3D网格。材质定义如何填充此图形的每个像素。

一个节点可以有任意数量的子节点，并且几何节点将被渲染，以便它们以子顺序出现，且父级位于其子级之后。

注意：这并未说明渲染器中的实际渲染顺序，仅保证视觉顺序。

有效的节点如下:
|节点名称|描述|
|--|--|
|QSGNode|“场景图”中所有节点的基类|
|QSGGeometryNode|用于“场景图”中所有可渲染的内容|
|QSGClipNode|“场景图”中实现“切割”功能|
|QSGOpacityNode|用来改变透明度|
|QSGTransformNode|实现旋转、平移、缩放等几何变换|

自定义节点通过继承`QQuickItem`类，重写`QQuickItem::updatePaintNode()`，并且设置 `QQuickItem::ItemHasContents` 标志的方式，添加到“场景图”。

警告：至关重要的是, 原生图形（OpenGL，Vulkan，Metal等）操作以及与“场景图”的交互只能在渲染线程上进行，主要在`updatePaintNode()`调用期间进行。

经验法则是仅在`QQuickItem::updatePaintNode()`函数内使用带有“QSG”前缀的类。

更多详细的信息，可以参考Qt文档： Scene Graph - Custom Geometry

### 预处理

节点具有虚函数`QSGNode::preprocess()`,该函数将在渲染“场景图”之前被调用。

节点子类可以设置标志`QSGNode::UsePreprocess`并重写`QSGNode::preprocess()`函数以对其节点进行预处理。

例如, 更新纹理的一部分, 或者将贝塞尔曲线划分为当前比例因子的正确细节级别。

### 节点所有权

节点的所有权归创建者，或者设置标志`QSGNode::OwnedByParent`后归“场景图”。

通常将所有权分配给“场景图”是可取的，因为这样可以简化“场景图”位于GUI线程之外时的清理操作。

### 材质

材质描述如何填充`QSGGeometryNode`中几何图形的内部。它封装了图形管线中顶点和片元阶段的着色器，并提供了足够的灵活性，

尽管大多数Qt Quick 项目本身仅使用了非常基本的材质，例如纯色和纹理填充。

想要对Qml中Item使用自定义着色的用户，可以直接在Qml中使用`ShaderEffect`。

下面是一个完整的材质类列表：

|材质名称|描述|
|--|--|
|QSGMaterial|封装了“着色器程序”的渲染状态|
|QSGMaterialRhiShader|表示独立于图形API的“着色器程序”|
|QSGMaterialShader|表示渲染器中的OpenGL“着色器程序”|
|QSGMaterialType|与`QSGMaterial`结合用作唯一类型标记|
|QSGFlatColorMaterial|“场景图”中渲染纯色几何图元的便捷方法|
|QSGOpaqueTextureMaterial|“场景图”中渲染不透明纹理几何图元的便捷方法|
|QSGTextureMaterial|“场景图”中渲染纹理几何图元的便捷方法|
|QSGVertexColorMaterial|“场景图”中渲染 逐顶点彩色图元的便捷方法|

更多详细的信息，可以参考Qt文档： Scene Graph - Simple Material 

### 便捷的节点

“场景图”API是一套 偏底层的接口，专注于性能而不是易用性。

从头开始编写自定义的几何图形和材质，即使是最基本的几何图形和材质，也需要大量的代码。

因此，API包含了一些节点类，以使最常用自定义节点的开发更便捷。

|节点名称|描述|
|--|--|
|QSGSimpleRectNode|`QSGGeometryNode`的子类，定义了矩形几何图元和纯色材质|
|QSGSimpleTextureNode|`QSGGeometryNode`的子类，定义了矩形几何图元和纹理材质|

## “场景图”和渲染

“场景图”的渲染发生在`QQuickWindow`类的内部，并且没有公共API可以访问它。

但是，渲染管线中有一些地方可以让用户附加应用程序代码。

可通过直接调用“场景图”使用的图形API(OpenGL、Vulkan、Metal等)来添加自定义“场景图”内容或插入

任意渲染命令。插入点由“渲染循环”定义。

有关“场景图”渲染器如何工作的详细说明，可以参考Qt文档: Qt Quick Scene Graph Default Renderer。

共有三种渲染循环变体: 基本渲染循环，窗口渲染循环和线程渲染循环。其中，基本渲染循环和窗口渲染循环是单线程的，

而线程渲染循环在专用线程上执行“场景图”渲染。

Qt尝试根据平台及可能使用的图形驱动程序选择合适的渲染循环。如果这不能满足你的需求，或者处于测试的目的，可以使用环境变量

`QSG_RENDER_LOOP`强制使用给定的渲染循环。要验证使用哪个渲染循环，请弃用`qt.scenegraph.general`日志类别。

注意：线程渲染循环和窗口渲染循环 依赖于图形API实现来进行节流，例如，在OpenGL环境下，“请求交换间隔”为1。一些图形驱动程序允许

用户忽略此设置并将其关闭，而忽略Qt的请求。在不阻塞“交换缓冲区”操作(或其它位置)的情况下，渲染循环将以尽快的速度运行动画并使

CPU 100%运转。如果已知系统无法提供基于`vsync`的限制,请通过设置环境变量`QSG_RENDER_LOOP = basic`使用 基本渲染循环。

### 线程渲染循环

在许多配置中，“场景图”将在专用渲染线程上进行。这样做是为了增加多核处理器的并行度，并更好地利用停顿时间。

这可以显著提高性能，但是与“场景图”进行交互的位置和时间加了一些限制。

以下是关于OpenGL环境下如何使用线程渲染循环的简单概述。除了OpenGL上下文的特定要求外，其它图形API的步骤也是相同的。

![](images\QtQuick\threadRenderLoop.png)

1. Qml场景中发生变化，触发调用`QQuickItem::update()`， 这可能是动画或者用户操作的结果。
   
   一个 `事件`会被`post`到渲染线程来启动新的一帧。

2. 渲染线程准备渲染新的一帧，GUI线程会启动阻塞。

3. 当渲染线程准备新的一帧时，GUI线程调用`QQuickItem::updatePolish()` 对场景中节点进行最终的“润色”,再渲染它们。

4. GUI 线程阻塞。

5. `QQuickWindow::beforeSynchronizing()`信号发出。应用程序可以对此信号进行直连(`Qt::DirectConnection`),
   
   以进行`QQuickItem::updatePaintNode()`之前所需的任何准备工作。

6. 将Qml状态同步到“场景图”中。自上一帧以来，所有已更改的节点上调用`QQuickItem::updatePaintNode()`函数完成同步。
   
   这是Qml与“场景图”中的节点唯一的交互。

7. GUI线程不再阻塞。

8. 渲染“场景图”：
   
    a. `QQuickWindow::beforeRendering()` 信号发出。应用程序可以直连(`Qt::DirectConnection`)此信号,来

     调用自定义图形API，然后将其可视化渲染在Qml场景之下。
      
    b. 指定了`QSGNode::UsePreprocess`标志的节点将调用其`QSGNode::preprocess()`函数。
    
    c. 渲染器处理节点。

    d. 渲染器生成状态并记录使用中的图形API的绘制调用。

    e. `QQuickWindow::afterRendering` 信号发出。应用程序可以直连(`Qt::DirectConnection`)此信号,来
    
    调用自定义图形API，然后将其可视化渲染在Qml场景之上。

    f. 新的一帧准备就绪。交换缓冲区(OpenGL)，或者记录当前命令，然后将命令缓冲区提交到图形队列(Vulkan,Metal)。

    `QQuickWindow::frameSwapped()`信号发出。

9. 渲染线程正在渲染时，GUI可以自由地进行动画、处理事件等。


当前默认情况下，线程渲染循环工作在 带opengl32.dll的Windows平台,具有Metal的MacOS平台，移动平台，

具有EGLFS的嵌入式Linux,以及平台无关的Vulkan，但这可能会有所改变。

通过在环境变量中设置`QSG_RENDER_LOOP=threaded`,可以强制使用线程渲染器。

### 非线程渲染循环 (基本渲染循环和窗口渲染循环)

当前默认在使用非线程渲染循环的环境，包括使用ANGLE及非默认opengl32实现的windows平台，使用OpenGL的MacOS，

以及一些特殊驱动的linux环境。这主要是一种预防措施，因为并非所有的OpenGL驱动和窗口系统的组合都经过测试。

同时，诸如ANGLE  或 Mesa llvmpipe之类的实现根本无法在线程渲染中正常运行。因此，对于这些环境，不能使用

线程渲染。

在MacOS OpenGL环境下，使用XCode 10 (10.14 SDK) 或更高版本进行构建时不支持线程渲染循环，因为这会选择在

MacOS 10.14上使用“基于图层的视图”。你可以使用XCode 9 (10.13 SDK)进行构建，以避开“基于图层的视图”,这种

情况下，线程渲染循环可以用并且默认会启用。Metal没有这样的限制。

非线程渲染循环默认在使用ANGLE的windows平台，而“基本渲染循环”用于其它需要非线程渲染循环的平台。

即使使用非线程渲染循环，也应像使用线程渲染循环一样编写代码，否则将使代码不可移植。

以下是非线程渲染循环中帧渲染序列的简化图示。

![](images\QtQuick\non-threadRenderLoop.png)

### 使用`QQuickRenderControl`自定义渲染控制

使用QQuickRenderControl时，驱动渲染循环的责任将转移到应用程序中。

在这种情况下，不使用内置的渲染循环。 

取而代之的是，由应用程序在适当的时候调用 `polish` `synchronize` `rendering`等渲染步骤,可以实现类似于上述

行为的线程渲染循环或非线程渲染循环。

### “场景图”和原生图形API的混合使用

“场景图”提供了两种方法，来集成应用程序提供的图形命令：

直接发出OpenGL、Vulkan、Metal等命令，以及在“场景图”中创建纹理化节点。


通过连接到`QQuickWindow::beforeRendering` 和 `QQuickWindow::afterRendering()`信号，应用程序可以直接在“场景图”

渲染的同一上下文中进行OpenGL调用。

使用Vulkan或者Metal之类的API，应用程序可以通过`QSGRendererInterface`来查询本机对象，例如“场景图”的命令缓冲区，

并在认为秖的情况下，向其记录命令。如信号的名称所示，用户随后可以在Qt Quick “场景图”下方或者上方渲染内容。

以这种方式集成的好处是不需要额外的帧缓冲区或者内存来执行渲染，并且消除了可能昂贵的纹理化步骤。缺点是Qt Quick 决定何时

调用信号，这是唯一允许OpenGL应用程序绘制的时间。

Qt提供了一些 “场景图”相关的示例，可在`examples`中找到
|例子名称|描述|
|--|--|
|Scene Graph - OpenGL Under QML|示例通过“场景图”的信号使用OpenGL|
|Scene Graph - Direct3D 11 Under QML|示例通过“场景图”的信号使用Direct3D|
|Scene Graph - Metal Under QML|示例通过“场景图”的信号使用Metal|
|Scene Graph - Vulkan Under QML|示例通过“场景图”的信号使用Vulkan|

另一个替代方式，是创建一个 `QQuickFrameBufferObject` (当前仅适用OpenGL)，在这个FBO内部渲染，然后将其

作为纹理显示在“场景图”中。

“Scene Graph - Rendering FBOs” 示例如何完成此操作。

还可以组合多个渲染上下文和多个线程以创建要在“场景图”中显示的内容。

“The Scene Graph - Rendering FBOs in a thread” 示例如何完成此操作。


“Scene Graph - Metal Texture Import”示例直接适用基础API创建和渲染纹理，然后在自定义`QQuickItem`中的

“场景图”中包装和适用此资源。该示例适用了Metal，但是概念也适用于所有其它图形API。

`QQuickFrameBufferObject`当前不支持，除OpenGL之外的其它图形API也可以采用这种方法。