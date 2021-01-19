---
title: 玩转Qml(15)-着色器效果ShaderEffect
photos: /images/ShaderEffect/7.png
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: 玩转Qml
abbrlink: 57407
date: 2019-06-22 13:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [关于文章](#%E5%85%B3%E4%BA%8E%E6%96%87%E7%AB%A0)
- [ShaderEffect](#ShaderEffect)
- [显示器如何显示色彩](#%E6%98%BE%E7%A4%BA%E5%99%A8%E5%A6%82%E4%BD%95%E6%98%BE%E7%A4%BA%E8%89%B2%E5%BD%A9)
- [GPU渲染流程](#GPU%E6%B8%B2%E6%9F%93%E6%B5%81%E7%A8%8B)
  - [渲染管线图](#%E6%B8%B2%E6%9F%93%E7%AE%A1%E7%BA%BF%E5%9B%BE)
  - [并行管线示意图](#%E5%B9%B6%E8%A1%8C%E7%AE%A1%E7%BA%BF%E7%A4%BA%E6%84%8F%E5%9B%BE)
- [着色器语言编码规范](#%E7%9D%80%E8%89%B2%E5%99%A8%E8%AF%AD%E8%A8%80%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83)
- [着色器代码示例](#%E7%9D%80%E8%89%B2%E5%99%A8%E4%BB%A3%E7%A0%81%E7%A4%BA%E4%BE%8B)
  - [示例](#%E7%A4%BA%E4%BE%8B)
  - [着色器代码](#%E7%9D%80%E8%89%B2%E5%99%A8%E4%BB%A3%E7%A0%81)
  - [顶点着色器](#%E9%A1%B6%E7%82%B9%E7%9D%80%E8%89%B2%E5%99%A8)
  - [片段着色器](#%E7%89%87%E6%AE%B5%E7%9D%80%E8%89%B2%E5%99%A8)
- [参考文献](#%E5%8F%82%E8%80%83%E6%96%87%E7%8C%AE)

## 简介

这次涛哥将会教大家一些ShaderEffect(参考QmlBook,译作：着色器效果)的相关知识。

前面的文章，给大家展示了进场动画，以及页面切换动画，大部分都使用了ShaderEffect，所以这次专门来说一下ShaderEffect。

## 源码

《玩转Qml》系列文章，配套了一个优秀的开源项目:TaoQuick

github https://github.com/jaredtao/TaoQuick

访问不了或者速度太慢，可以用国内的镜像网站gitee

https://gitee.com/jaredtao/TaoQuick

## ShaderEffect

动画只能控制组件的属性整体的变化，做特效需要精确到像素。

Qml中提供了ShaderEffect这个组件，就能实现像素级别的操作。

ShaderEffect允许我们在Qml的渲染引擎SceneGraph上，利用强大的GPU进行渲染。

使用ShaderEffect，需要有一些图形学知识，了解GPU渲染管线，了解图形API如OpenGL、DirectX等，同时也需要一些数学知识。

图形学的知识体系还是非常庞大的，要系统的学习，需要看很多书籍。入门级的比如“红宝书”《OpenGL编程指南》、“蓝宝书”《OpenGL超级宝典》......

一篇文章是说不完的，涛哥水平也有限。所以本文从实用的角度出发，按照涛哥自己的理解，提炼一些必要的知识点，省略一些无关的细节，

让各位Qt开发者能了解GPU原理，能看懂、甚至于自己写一些简单的着色器代码，就大功告成了。说的不对的地方，也欢迎大佬来指点。

## 显示器如何显示色彩

先来了解一下，显示器是如何显示出各种色彩的。

假如我们把显示器的画面放大100倍，就会看到很多整齐排列的像素点。

![](/images/ShaderEffect/1.png)

继续放大，就会发现每个像素点，由三种发光的元件组成，这三种元件分别发出红、绿、蓝三种颜色的光。三种颜色的光组合在一起，

就是人眼看到的颜色。这就是著名的RGB颜色模型。

![](/images/ShaderEffect/2.png)

如果把这三种光的亮度分为255个等级，就能组合出16777216种不同颜色的光。

GPU的任务，就是通过计算，给出每一个像素的红、绿、蓝 （简称r g b）三种颜色的数值，让显示器去"发出相应的光"。

(这样说可能不太严谨、不太专业，只是方便大家理解。另一方面，本文的目的，

是让大家学习如何写特效，不是去造显卡/造显示器。所以请专业人士见谅！)

注：参考[1]

## GPU渲染流程

我们以画一个填充色的三角形为例，来说明

![](/images/ShaderEffect/3.png)

### 渲染管线图

下图是一个简易的渲染管线，引用自 [LearnOpenGL](https://learnopengl-cn.readthedocs.io/zh/latest/)

![](/images/ShaderEffect/5.png)

画一个三角形，要经历顶点着色器、图元装配、几何着色器、片段着色器、光栅化等阶段。

其中蓝色部分是可以自定义的，自定义是指，按照图形API规范，写一段GPU能编译、运行的代码。

(这种代码就是着色器代码。可以自定义的这种渲染管线，就是可编程渲染管线，与之相对的是古老的固定渲染管线。)

这里各个阶段，分别引用一下，LearnOpenGL中的介绍(看不懂可以先跳过，看我画的图)：

```
1 管线的第一个部分是顶点着色器(Vertex Shader)，它把一个单独的顶点作为输入。顶点着色器主要的目的是

把3D坐标转为另一种3D坐标（后面会解释），同时顶点着色器允许我们对顶点属性进行一些基本处理。

2 图元装配(Primitive Assembly)阶段将顶点着色器输出的所有顶点作为输入（如果是GL_POINTS，那么就是一个顶点），

并所有的点装配成指定图元的形状；本节例子中是一个三角形。

3 图元装配阶段的输出会传递给几何着色器(Geometry Shader)。几何着色器把图元形式的一系列顶点的集合作为输入，

它可以通过产生新顶点构造出新的（或是其它的）图元来生成其他形状。例子中，它生成了另一个三角形。

4 几何着色器的输出会被传入光栅化阶段(Rasterization Stage)，这里它会把图元映射为最终屏幕上相应的像素，

生成供片段着色器(Fragment Shader)使用的片段(Fragment)。在片段着色器运行之前会执行裁切(Clipping)。

裁切会丢弃超出你的视图以外的所有像素，用来提升执行效率。

5 片段着色器的主要目的是计算一个像素的最终颜色，这也是所有OpenGL高级效果产生的地方。通常，片段着色器包

含3D场景的数据（比如光照、阴影、光的颜色等等），这些数据可以被用来计算最终像素的颜色。
```

### 并行管线示意图

概念还是挺多的，而且很多教程都有渲染管线图。但是涛哥觉得，对于我们开发Shader来说，一定要有并行的意识，然而大部分

管线图，都没有体现出GPU的并行特性。所以涛哥自己画了一个草图：

![](/images/ShaderEffect/6.png)

解释一下吧，CPU传入了3个顶点到GPU，GPU将这三个顶点，传递给三个顶点着色器。

这里要意识到，顶点着色器开始，就是并行处理了。GPU是很强大的SIMD架构（单指令流多数据流）。

如果我们自定义了一段顶点着色器代码，则三个顶点会同时运行这段代码。（后面的片段着色器代码，就是N个点同时运行）

顶点着色器进行处理，传递给图元装配。

图元装配阶段，进行了顶点扩充，变成N个点，N看作三角形面积所在的点。

之后N个点依次传给 几何着色器->光栅化->片段着色器，最后经过测试与混合后，输出到屏幕。

可以自定义编程的，有顶点着色器、几何着色器、片段着色器（有的地方也叫像素着色器），顺带提一下，还有另外三种：

曲面控制着色器、曲面评估着色器 和 计算着色器。

一般我们的关注点，都会在片段着色器上。涛哥之前写的12种特效，就只用了自定义的片段着色器。

著名的ShaderToy网站，也是只关注片段着色器。[ShaderToy](https://www.shadertoy.com)

## 着色器语言编码规范

我们可以把着色器语言，当作运行在GPU上的C语言。

Qt的ShaderEffect支持的着色器语言包括OpenGL规范中的GLSL，和DirectX规范中的HLSL，这两种着色语法上有些细微的区别，但是可以互相转换。

我们就以glsl为主。详细的语言规范，在khronos的官网, 各个版本都有: https://www.khronos.org/registry/OpenGL/specs/gl/

![](/images/ShaderEffect/8.png)

桌面版 OpenGL 版本众多，而嵌入式系统也有专用的OpenGL ES。

安卓手机、平板设备一般就是OpenGL ES，新的设备都支持ES 3.0，老的设备一般只支持到ES 2.0

OpenGL ES 的语言规范文档在这里： https://www.khronos.org/registry/OpenGL/specs/es/2.0/

我们就用Qt默认的版本。

## 着色器代码示例

### 示例

这里用Qt帮助文档中的示例代码，来说明。

```qml
  import QtQuick 2.0

  Rectangle {
      width: 200; height: 100
      Row {
          Image { id: img;
                  sourceSize { width: 100; height: 100 } source: "qt-logo.png" }
          ShaderEffect {
              width: 100; height: 100
              property variant src: img
              vertexShader: "
                  uniform highp mat4 qt_Matrix;
                  attribute highp vec4 qt_Vertex;
                  attribute highp vec2 qt_MultiTexCoord0;
                  varying highp vec2 coord;
                  void main() {
                      coord = qt_MultiTexCoord0;
                      gl_Position = qt_Matrix * qt_Vertex;
                  }"
              fragmentShader: "
                  varying highp vec2 coord;
                  uniform sampler2D src;
                  uniform lowp float qt_Opacity;
                  void main() {
                      lowp vec4 tex = texture2D(src, coord);
                      gl_FragColor = vec4(vec3(dot(tex.rgb,
                                          vec3(0.344, 0.5, 0.156))),
                                               tex.a) * qt_Opacity;
                  }"
          }
      }
  }

```
这段代码的效果是

![](/images/ShaderEffect/7.png)

左边是本来的绿色的Qt的logo，右边是处理过后的灰色logo。

### 着色器代码
ShaderEffect的vertexShader属性就是顶点着色器了，其内容是一段字符串。按照着色器规范实现的。

同样的，fragmentShader属性 即片段着色器。

我们能在着色器中看到void main函数，这个便是着色器代码的入口函数，和C语言很像。

在main之前，还有一些全局变量,我们逐条来说明一下

在顶点着色器中，有这三种不同用处的变量：uniform、attribute、varying。

这些变量的值都是从CPU传递过来的。

如果你写过原生OpenGL的代码，就会知道，其中很大一部分工作，就是在处理CPU数据传递到GPU着色器中。

而Qml的ShaderEffect简化了这些工作，只要写一个property，名字、类型和着色器中的对应上，就可以了。

### 顶点着色器

```glsl
attribute highp vec4 qt_Vertex; 
```
attribute是"属性"变量，按照前面涛哥画的管线图来说，三个顶点着色器同时运行时，每个着色器中

的attribute值都不一样。这里的qt_Vertex，可以理解为分别是三角形的三个顶点。

highp是精度限定符，这里先忽略，具体细节可以参考语言规范文档。后面的lowp、 medium也是精度限定符。

vec4就是四维向量，类似QVector4D。

qt_Vertex是变量的名字。

这条语句的作用，就是声明一个用来存储顶点的attribute变量qt_Vertex。

uniform是统一变量，三个顶点着色器同时运行时，它们取得的uniform变量值是一样的。

varying表示这个顶点着色器的输出数据，将传递给后面的渲染管线。

```
void main() 
{
    coord = qt_MultiTexCoord0;
    gl_Position = qt_Matrix * qt_Vertex;
}
```
这段main函数，将CPU传进来的纹理坐标qt_MultiTexCoord0数据，通过varying变量coord，传递给了下一个阶段，然后使用矩阵进行了坐标转换，

并将结果存储在glsl的内置变量gl_Position中。

### 片段着色器

片段着色器中，就没有attribute了。uniform是一样的统一变量，varying是上一个阶段传递进来的数据。

```glsl
uniform sampler2D src;
```
sampler2D是二维纹理。所谓纹理嘛，可以理解成一张图片，一个Image。

src这个变量，就代表外面传进来的那个Image。 sampler2D也可以是任意可视的Item(通过ShaderEffectSource传递进来)

来看一下main函数

```
void main() 
{
    lowp vec4 tex = texture2D(src, coord);
    gl_FragColor = vec4(vec3(dot(tex.rgb,vec3(0.344, 0.5, 0.156))), tex.a) * qt_Opacity;
}
```
这里使用了纹理
```glsl
lowp vec4 tex = texture2D(src, coord);
```

texture2D是一个内置函数，专业术语叫“对纹理进行采样”，什么意思呢？

假如coord的值是(0,0),那就是对src指代的这张图片，取x=0、y=0的坐标点的像素，作为返回值，存储在tex变量中。

这里注意一下纹理坐标的取值范围。假如Qml中图片的大小是100x100，其取值范围从(0, 0) -> (100, 100)

这里的传进来的纹理坐标，取值范围是(0, 0) -> (1, 1) ，GPU为了方便计算，都进行了归1化处理。将范围缩小到0 - 1

```glsl
 gl_FragColor = vec4(vec3(dot(tex.rgb, vec3(0.344, 0.5, 0.156) )), tex.a) * qt_Opacity;
```
dot(tex.rgb, vec3(0.344, 0.5, 0.156) ) 是对两个三维向量进行了点乘。

tex.rgb是GLSL中的取值器语法。 tex是一个四维变量，可以用tex.r tex.g tex.b tex.a分别取出其中一维，也可以任意两个组合、三个

组合取值。 

rgba可以取值，xyzw也可以取值， stpq也行，但只能三种选一种，不能混用。

vec4（vec3(), tex.a) 是用三维向量再加一个变量，构造四维向量。

这条语句其实是一个RGB转灰度的公式，可以自行搜索相关的资料。

gl_FragColor 是内置变量，表示所在片段着色器的最终的输出颜色。

## 参考文献

[1] https://zhuanlan.zhihu.com/p/43467096

[2] https://learnopengl-cn.github.io/