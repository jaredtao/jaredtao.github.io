---
title: 玩转QtQuick(2)-默认渲染器
photos: /images/qt_extended_48x48.png
tags:
  - Qt
  - Qml
  - QtQuick
categories: 玩转QtQuick
abbrlink: 18621
date: 2021-01-21 13:44:23
---
- [简介](#简介)
- [Qt Quick的默认渲染器](#qt-quick的默认渲染器)
  - [批次渲染](#批次渲染)
    - [不透明图元](#不透明图元)
    - [Alpha混合图元](#alpha混合图元)
    - [混合3D图元](#混合3d图元)
    - [纹理集](#纹理集)
  - [批次渲染的根节点](#批次渲染的根节点)
    - [变换的节点](#变换的节点)
    - [裁剪](#裁剪)
    - [顶点缓冲](#顶点缓冲)
  - [抗锯齿](#抗锯齿)
    - [顶点抗锯齿](#顶点抗锯齿)
    - [多采样抗锯齿](#多采样抗锯齿)
  - [性能](#性能)
  - [可视化](#可视化)
    - [批次可视化](#批次可视化)
    - [裁剪可视化](#裁剪可视化)
    - [变更可视化](#变更可视化)
    - [OverDraw 过渡绘制可视化](#overdraw-过渡绘制可视化)
  - [通过QtRHI(硬件渲染接口) 进行渲染](#通过qtrhi硬件渲染接口-进行渲染)

# 简介

这是《玩转QtQuick》系列文章的第二篇，主要是介绍Qt Quick的默认渲染器。

****本文会涉及到一些图形学的基本概念，例如：材质、纹理、光栅化、图元等，建议参考相关资料，本文不做进一步的解释。****

因为Qt官方文档写的比较全面，所以本文主要是对官方文档的翻译，同时会补充一些个人理解。

翻译主要参考Qt5.15的文档，适当做了一些调整，尽量信达雅，尽量说人话。

下面翻译开始

# Qt Quick的默认渲染器

本文介绍默认渲染器在内部的工作方式，以方便开发者们以最佳的方式

使用它(编写代码)，包括性能和功能。

通常无需了解渲染器的内部结构，就能够获得良好的性能。

但是，在与场景图集成或弄清楚为什么无法从图形芯片中挤出最大效率时，这可能会有所帮助。

(即使在每个帧都是唯一的并且所有内容都是从头开始上传的情况下，默认渲染器也将表现良好)

Qml场景中的`Item`将填充`QSGNode`实例树。一旦实例树创建好之后，此树将完整描述如何渲染特定的帧。

它不会包含对任何`Item`的反向引用，并且在大多数平台上将通过单独的线程进行处理和渲染。

渲染器是“场景图”的自包含部分，它遍历`QSGNode`树，并使用`QSGGeometryNode`中定义

的几何形状和`QSGMaterial`中定义的着色器状态来更新图形状态并生成DrawCall。

如果有需要，可以使用内部的“场景图”后端API完全替换渲染器。

对于希望利用非标准硬件功能的平台供应商来说，这最为有趣。

对于大多数用例，默认渲染器就足够了。

默认渲染器着重于优化渲染的两种主要策略：批量处理调用和在GPU上保留几何图元。

## 批次渲染

传统的2D API (例如QPainter，Cairo或者Context2D)被设计为每帧处理大量单独的DrawCall,而当DrawCall的次数

非常少且状态更改保持在一定水平时，OpenGL和其它硬件加速的API表现最佳。

考虑以下用例：

![](/images/QtQuick/batching.png)

绘制此列表的最简单方法是逐行进行。

首先，绘制背景。背景是特定颜色的矩形。在OpenGL术语中，这意味着使用一个着色器程序进行纯色填充，设置填充颜色，

设置包含x和y偏移量的转换矩阵，然后使用例如`glDrawArrays`绘制组成矩形的两个三角形。

接下来绘制图标。用OpenGL术语来说，这意味着使用一个着色器程序来绘制纹理，激活要使用的纹理，设置转换矩阵，启用alpha混合，

然后使用例如`glDrawArrays`绘制组成图标边界矩形的两个三角形。

行 之间的文本和分隔线遵循类似的模式。

对于列表中的每一行都重复此过程，因此对于更长的列表，OpenGL状态变更和DrawCall所带来的

开销完全超过了使用硬件加速API所能提供的好处。

当每个图元都很大时，此开销可以忽略不计，但是在典型的UI环境中，有许多小项加起来会产生相当大的开销。

默认的“场景图”渲染器也在这些限制内运行，并且会尝试将单个图元合并到批次中，同时保留完全相同的视觉效果。

结果是更少的OpenGL状态变更和最少的DrawCall调用，从而实现了最佳性能。

### 不透明图元

渲染器将不透明图元和需要透明度的图元进行了分类。

通过使用OpenGL的Z缓冲为每一个图元赋予唯一的z值，渲染器可以自由地对不透明图元进行重新排序，而无需考虑

它们在屏幕上的位置以及与它们重叠的其它元素。通过查看每个图元的材质状态，渲染器将创建不透明的批次渲染。

在QtQuick的主要Item中，属于不透明图元的包括`不透明颜色的Rectangle`和`完全不透明的Image`,主要是`JPEG`和`BMP`格式。

使用不透明图元的另一个好处是，不透明图元不需要启用`GL_BLEND`，这个操作可能会非常耗性能，尤其是在移动端和嵌入式GPU上。

不透明图元在启用`glDepthMask`和`GL_DEPTH_TEST`的情况下以从前到后的方式渲染。在内部进行`early-z`校验的GPU上，这意味着

片元着色器不需要针对被遮盖的像素或像素块运行。

请注意，渲染器仍需要考虑这些节点，并且顶点着色器仍将为这些图元的每个顶点运行，因此，如果应用程序知道某些东西被完全遮盖，则

最好的办法是设置`Item::visible`或`Item::opacity`隐藏它。

`Item::z`用来控制Item相对于其同级元素的“堆叠顺序”，它与渲染器和OpenGL的z缓冲没有直接关系。

### Alpha混合图元

一旦绘制了不透明的图元，渲染器将禁用`glDepthMask`,启用`GL_BLEND`并以从后到前的方式渲染所有alpha混合图元。

alpha混合图元的批次渲染在渲染器内需要更多的工作，因为重叠的元素需要以正确的顺序进行渲染，以使alpha看起来正确。

仅仅依靠Z缓冲是不够的。渲染器在所有alpha混合图元上进行传递，除了其材质状态外，还将查询其边框，以确定哪些元素

可以批次渲染，哪些元素不能批次。

![](/images/QtQuick/batching2.png)

上图左边的情况，可以在一次DrawCall中渲染蓝色背景，而在另一次DrawCall中渲染两个文本元素，因为这些文本仅与其

同一层的背景重叠。

右边的情况，`Item 4`的背景覆盖了`Item 3`的背景，因此每一个背景和文本需要在不同的DrawCall中渲染。

在Z方向上，alpha节点与不透明节点交错，且在可用时触发`early-z`。同样的，将`Item::visible`设置为false会快很多。

### 混合3D图元

“场景图”支持伪3D和适当的3D图元。

例如，可以用`ShaderEffect`来实现“页面卷曲”效果，或者可以使用`QSGGeometry`和自定义材质来实现凹凸贴图。实现这

些功能时，开发者需要意识到默认渲染器已经使用了深度缓冲区。

渲染器修改了`QSGMaterialShader::vertexShader()`返回的顶点着色器，并在应用了模型视图和投影矩阵之后压缩了

顶点的z值，然后在z上添加了一个小平移以将其放置在正确的z位置。

压缩时会假定z值在0到1的范围内。

### 纹理集

激活的纹理在OpenGL中是个唯一的状态，这意味着使用不同纹理的多个图元无法批次渲染。因此，Qt Quick“场景图”允许

将多个`QSGTexture`实例分配为较大纹理的较小子区域，也就是“纹理集”。

纹理集的最大好处是多个`QSGTexture`实例引用同一个OpenGL纹理实例。这样还可以批量处理带纹理的DrawCall，例如

Qml中的`Image`,`BorderImage`,`ShaderEffect`等，以及C++中的`QSGSimpleTextureNode`和自定义

的`QSGGeometryNode`都使用了纹理。

尺寸过大的纹理不会进入纹理集。

纹理集使用带参数`QQuickWindow::TextureCanUseAtlas`的函数调用`QQuickWindow::createTextureFromImage()`创建。

纹理集没有范围从0到1的坐标。使用`QSGTexture::normalizedTextureSubRect()`获取纹理坐标。

“场景图”使用试探法来确定纹理集应该多大以及输入大小的阈值。如果需要不同的值，可以通过设置环境变量

`QSG_ATLAS_WIDTH=[width]`, `QSG_ATLAS_HEIGHT=[height]`和`QSG_ATLAS_SIZE_LIMIT=[size]`来覆盖试探法。

对于平台供应商而言，更改这些数值通常是有趣的。

## 批次渲染的根节点

除了将兼容的图元合并到一个批次，默认渲染器还尝试将每帧需要发送到GPU的数据量减到最少。

默认渲染器会标记在一起的子树，并尝试将它们放入单独的批次中。

识别批次后，即可使用`顶点缓冲对象`将其合并，上传并存储在GPU内存中。

### 变换的节点

每个QtQuick中的`Item`会往场景树中插入一个`QSGTransformNode`来管理其x、y坐标、缩放比例。`子Item`会

附加在此变换节点之下。默认渲染器会跟踪帧之间变换节点的状态，并将查看子树以确定：变换节点作为一个

批次渲染的根节点是否良好。在帧之间变化且具有相当复杂的子树的变换节点可以成为批次渲染的根节点。

批次渲染根节点的子树中`QSGGeometryNodes`相对于CPU上的根节点已经预先转换过了，然后讲它们上传

并保留在GPU上。当变换发生时，渲染器仅需要更新根节点的矩阵，而无需更新每个单独的节点，从而使列表

和网格滚动非常快。对于连续的帧，只要不添加或删除节点，就可以快速地、不增加消耗地渲染。当新内容

进入子树时，将对其进行重建，但这仍然相当较快。

在`Grid`或`List`中滚动时，会有节点添加或者删除，但也总会有一些帧是不变的。

****

将变换节点 标记为 批次根节点的另一个好处是，它允许渲染器保留树中未更改的部分。

例如：UI由一个`List`和一行按钮 组成。滚动`List`并添加或删除`Delegate`时，UI的其余

部分(一行按钮)保持不变,可以使用存储在GPU上的几何图元进行绘制。

可以使用环境变量`QSG_RENDERER_BATCH_NODE_THRESHOLD=[count]`和`QSG_RENDERER_BATCH_VERTEX_THRESHOLD=[count]`来

覆盖要成为批次根节点的转换节点和顶点阈值。覆盖这些标志对平台供应商最有用。

在批次渲染根节点之下，会为每个唯一的材质状态集和几何图元类型创建一个批次节点。


### 裁剪

将`Item::clip`设置为true时，将创建一个`QSGClipNode`,其几何形状为矩形。

默认渲染器将通过在OpenGL中使用`scissoring`来应用此裁剪操作。如果将`Item`

旋转了非90°角，则使用OpenGL的模版缓冲区。QtQuick的`Item`仅支持通过Qml启用

矩形的裁剪，“场景图”API和渲染器则支持任何形状的裁剪。

将裁剪应用于子树时，该子树需要使用唯一的OpenGL状态进行渲染。这意味着当`Item::clip`

为true时，该`Item`的批次渲染仅限于其`子Item`。当有许多子级(例如`ListView`或`GridView`)或

复杂的子级(例如`TextArea`)时，这是好事。

应该避免在较小的`Item`上使用裁剪，因为它会阻止批次渲染。这包括`Button`上面的`Label`，`TextField`和`Table`中的`Delegate`。

### 顶点缓冲

每个批次渲染都会使用顶点缓冲区对象(VBO)将其数据存储在GPU上。该顶点缓冲区保留在帧之间，并在“场景图”所

表示的部分发生更改是更新。

默认情况下，渲染器将使用`GL_STATIC_DRAW`将数据上传到VBO。

通过环境变量`QSG_RENDERER_BUFFER_STRATEGY=[strategy]`可以选择其它上传策略，有效的策略还包括`stream`和`dynamic`。

更改此值对平台供应商最有用。

## 抗锯齿

“场景图”支持两种类型的抗锯齿。

默认情况下，诸如`Rectangle`和`Image`之类的图元，将通过沿图元的边缘添加顶点的方式，使

边缘淡化到透明，以实现抗锯齿。我们称此方法为顶点抗锯齿。

如果用户通过`QQuickWindow::setFormat()`将`QSurfaceFormat`设置为大于0的值，请求OpenGL多重采样，“场景图”将首选

基于多重采样的抗锯齿(MSAA)。

这两种技术将影响渲染器的内部实现方式，并且具有不同的限制。

通过设置环境变量`QSG_ANTIALIASING_METHOD `为`msaa`或者`vertex`也可以覆盖使用的抗锯齿方法。

即使两个图元的边在数学上相同，顶点抗锯齿也会在相邻图元的边缘之间产生接缝。多重采样抗锯齿则不会如此。

### 顶点抗锯齿

可以使用`Item::antialiasing`属性启用和禁用单个`Item`的顶点抗锯齿。在硬件支持的前提下，无论是正常渲染的图元，还是捕获到

帧缓冲区对象中的图元(例如使用`ShaderEffectSource`),顶点抗锯齿都可以正常运行并产生更高质量的抗锯齿功能。

使用顶点抗锯齿的不利之处在于，每个启用了抗锯齿的图元都必须进行混合。在批次渲染方面，这意味着渲染器需要做更多的工作来确定

图元是否可以进行批次渲染。如果和场景中其它元素重叠，也可能导致更少的批次渲染，从而影响性能。

在低端硬件上，混合操作也可能会非常耗性能。对于覆盖屏幕大部分区域的图像或者圆角矩形，这些图元内部所需要的混合操作数量

可能会导致严重的性能损耗，因为必须混合整个图元。

### 多采样抗锯齿

多采样抗锯齿是一项硬件功能，其中硬件会计算图元中每个像素的覆盖值。一部分硬件可以以非常低的成本进行多次采样，而另一些

硬件需要更多的内存和GPU周期来渲染一帧，

使用多采样可以对许多图元进行抗锯齿（例如圆角矩形和图片），并且在“场景图”中仍然是不透明的。这意味着在创建渲染批次时，

渲染器的工作会更加轻松，并且可以依赖`early-z`来避免过渡渲染。

使用多重采样抗锯齿时，渲染到帧缓冲区对象中的内容需要额外的扩展以支持帧缓冲区的多重采样。通常是`GL_EXT_framebuffer_multisample `

和`GL_EXT_framebuffer_blit`。大多数台式机芯片都具有这些扩展，但是在嵌入式芯片中却很少见。

如果硬件中不提供帧缓冲区多采样，则不会进行多采样抗锯齿，包括`ShaderEffectSource`。

## 性能

如文章开头所说，不需要了解渲染器的详细信息就能够获得良好的性能。默认渲染器在设计时就针对常见用例进行了优化，

并且在几乎任何情况下都将表现良好。

* 有效的批次渲染可带来良好的性能，并尽可能少地上传几何图形。通过设置环境变量`QSG_RENDERER_DEBUG=render`，渲染
  
  器将输出相应的统计信息，包括：批次渲染进行的程度，使用的批次数量，保留的批次及不透明和透明的批次数量等。
  
  追求最佳性能时，应仅在真正需要时上传数据，批次数量应该少于10个，且至少3-4个不透明批次。

* 默认渲染器不执行任何CPU端的视口裁剪或遮挡检测。如果某些内容不可见，则不应该显示，使用`Item::visible`将其设置

  为false。不添加这样的逻辑的主要原因是，它增加了额外的成本，这也将损害那些表现良好的应用程序。

* 确保纹理集被使用。除非图像特别大，否则`Image`和`BorderImage`  将使用它。C++代码中想要创建纹理集，需在调用
 
  `QQuickWindow::createTexture()`时传递`QQuickWindow::TextureCanUseAtlas`参数。通过设置环境变量
  
  `QSG_ATLAS_OVERLAY`，所有纹理集将被着色,以便在应用程序中轻松识别它们。

* 尽可能使用不透明图元。不透明图元在渲染器中处理速度更快，在GPU上绘制速度更快。例如，即使每个像素都是不透明的，PNG

  文件也会经常具有alpha通道。JPG文件始终是不透明的。当图像提供给`QQuickImageProvider`或者使用

  `QQuickWindow::createTextureFromImage()`创建图像时，请尽可能使用`QImage::Format_RGB32`格式。

* 如前文所示，重叠的`复合Item`无法批次渲染。

* 裁剪会中断批次渲染。切勿在表格内的单元格，`delegate`或者类似的元素中使用裁剪。使用`省略`代替文本裁剪。

创建一个返回裁剪后图像的`QQuickImageProvider`，代替图像裁剪。

* 批次渲染仅适用于16位索引。所有QtQuick内置的`Item`都使用了16位索引，但是自定义几何图元也可以自由使用32位索引。

* 一些材质的标志会阻止批次渲染，其中最受限制的一个是`QSGMaterial::RequiresFullMatrix`,它阻止了所有批次渲染。

* 具有单色背景的应用程序应使用`QQuickWindow::setColor()`而不是顶级带颜色的`Rectangle`。
  
 `QQuickWindow::setColor()`将在`glClear()`的调用中使用，这是比较快的。

*  生成`Mipmap`的`Image`不会放在纹理集中，也不会进行批次渲染。

* 存在一个OpenGL驱动程序相关的`Bug`：帧缓冲对象(FBO)回读时发生的一些错误会损坏渲染的字形。如果在环境变量

中设置`QML_USE_GLYPHCACHE_WORKAROUND `，则Qt会在`RAM`中保留该字形的其它副本。这意味着当渲染以前

未渲染的字形时，性能会稍低，因为Qt通过CPU访问额外的副本。这也意味着字形缓存将使用两倍的内存。渲染质量不受影响。

如果应用程序性能不佳，需要确认瓶颈是否在渲染。此时可以使用探查器`Profiler`! 设置环境变量`QSG_RENDER_TIMING=1`

将输出许多有用的时序参数，这些参数可以用来查明问题所在。


## 可视化

为了可视化“场景图”默认渲染器的各个方面，可以将`QSG_VISUALIZE`环境变量设置为下面每个部分中详细介绍的值其中之一。

下面这段Qml代码提供了一些变量输出的示例：

```qml
 import QtQuick 2.2

 Rectangle {
     width: 200
     height: 140

     ListView {
         id: clippedList
         x: 20
         y: 20
         width: 70
         height: 100
         clip: true
         model: ["Item A", "Item B", "Item C", "Item D"]

         delegate: Rectangle {
             color: "lightblue"
             width: parent.width
             height: 25

             Text {
                 text: modelData
                 anchors.fill: parent
                 horizontalAlignment: Text.AlignHCenter
                 verticalAlignment: Text.AlignVCenter
             }
         }
     }

     ListView {
         id: clippedDelegateList
         x: clippedList.x + clippedList.width + 20
         y: 20
         width: 70
         height: 100
         clip: true
         model: ["Item A", "Item B", "Item C", "Item D"]

         delegate: Rectangle {
             color: "lightblue"
             width: parent.width
             height: 25
             clip: true

             Text {
                 text: modelData
                 anchors.fill: parent
                 horizontalAlignment: Text.AlignHCenter
                 verticalAlignment: Text.AlignVCenter
             }
         }
     }
 }
```

左侧的`ListView`,我们将其`clip`属性设置为true。右侧的`ListView`，我们将每个`delegate`的clip属性设置为true。

以此来说明裁剪对批次渲染的影响。

这是正常的运行结果

![](/images/QtQuick/batch1.png)

可视化元素不考虑裁剪，并且渲染顺序是任意的。

### 批次可视化

设置环境变量`QSG_VISUALIZE`为`batches`可以在渲染器中可视化查看批次。

合并过的批次以纯色渲染，未合并的批次以对角线图案渲染。独立的颜色越少意味着批次分配的越好。

如果未合并的批次中包含许多单独的节点，则是比较糟糕的。

![](/images/QtQuick/batch2.png)

QSG_VISUALIZE=batches

### 裁剪可视化

设置环境变量`QSG_VISUALIZE `为`clip`,渲染器会在“场景图”中渲染红色区域来指示裁剪区域。

默认情况下`Item`不裁剪，因此不会显示裁剪区域。

![](/images/QtQuick/batch3.png)

QSG_VISUALIZE=clip

### 变更可视化

设置环境变量`QSG_VISUALIZE `为`changes`,可以在“场景图”中看到变更。“场景图”中的变更以随机颜色的

闪烁叠加显示。图元上的变更以纯色显示，而批次渲染根节点的变更以特定的`pattern`显示。

### OverDraw 过渡绘制可视化

设置环境变量`QSG_VISUALIZE `为`overdraw `,可以在“场景图”中看到过渡绘制。可视化的3D视图中，

所有过渡绘制的`Item`会高亮显示。此模式也可以用来检测视口之外的几何图元。不透明的`Item`以绿色显示，

而半透明的`Item`以红色显示。视口的边框为蓝色显示。不透明的内容使“场景图”更易于处理，渲染速度更快。
 
请注意，上面的代码中顶层矩形框`Rectangle`是多余的，因为窗口也是白色的，在这种情况下渲染矩形框会造成资源浪费。

将其更改为`Item`会略微提高性能。

![](/images/QtQuick/batch4.png)

QSG_VISUALIZE=overdraw

## 通过QtRHI(硬件渲染接口) 进行渲染

从Qt5.14开始，默认适配层增加了一个选项，可以使用 QtGui模块提供的图形抽象层Qt Rendering Hardware Interface (RHI)

进行渲染。启用后，将不进行OpenGL调用，而是使用抽象层提供的API来渲染“场景图”，然后将其转换为OpenGL,

Vulkan,Metal或者Direct3D调用。

通过一次编写着色器代码，编译为SPIR-V,然后转换为适用于各种图形API的语法，实现着色器的统一处理。

要启用此功能，代替直接OpenGL调用，可以通过下面的变量:

|环境变量|有效值|描述|
|--|--|--|
|QSG_RHI|1|启用通过RHI的渲染。除非被`QSG_RHI_BACKEND`覆盖，否则将根据平台选择目标图像API。默认值为window平台使用Direct3D 11, MacOS平台使用metal，其它平台使用OpenGL|
|QSG_RHI_BACKEND|vulkan,metal,opengl,d2d11|请求使用指定的图形API|
|QSG_INFO|1|与基于OpenGL的渲染路径一样，设置此选项将在初始化Qt Quick“场景图”时启用打印信息。 这对于故障排除非常有用。|
|QSG_RHI_DEBUG_LAYER|1|在适用的情况下（Vulkan，Direct3D），启用图形API实现的调试或验证层（如果有）。|
|QSG_RHI_PREFER_SOFTWARE_RENDERER|1|请求使用软光栅化的适配器或物理设备。仅在API支持枚举适配器(Direct3D或Vulkan)时适用，否则会被忽略|


希望始终使用单个指定的图形API运行应用程序，也可以通过C++代码来设置。

例如，在构造任何`QQuickWindow`之前，在main函数的早期进行以下调用将强制使用vulkan
```C++
 QQuickWindow::setSceneGraphBackend(QSGRendererInterface::VulkanRhi);
```

可以查看`QSGRendererInterface::GraphicsApi`文档。以Rhi结尾的枚举值等价于设置`QSG_RHI `和`QSG_RHI_BACKEND `。

除非被`QSG_RHI_PREFER_SOFTWARE_RENDERER `或特定后端的变量（例如`QT_D3D_ADAPTER_INDEX` 或者 `QT_VK_PHYSICAL_DEVICE_INDEX`）覆盖，

否则所有QRhi后端都会选择系统默认的GPU适配器或物理设备。目前没有进一步的适配器相关配置项。