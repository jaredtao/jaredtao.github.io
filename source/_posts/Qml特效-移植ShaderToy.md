---
title: Qml特效-移植ShaderToy
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 特效
categories: Qml特效
abbrlink: 9836
date: 2019-07-04 13:44:23
---

- [简介](#%e7%ae%80%e4%bb%8b)
- [关于文章](#%e5%85%b3%e4%ba%8e%e6%96%87%e7%ab%a0)
- [效果预览](#%e6%95%88%e6%9e%9c%e9%a2%84%e8%a7%88)
  - [穿云洞](#%e7%a9%bf%e4%ba%91%e6%b4%9e)
  - [星球之光](#%e6%98%9f%e7%90%83%e4%b9%8b%e5%85%89)
  - [蜗牛](#%e8%9c%97%e7%89%9b)
  - [超级马里奥](#%e8%b6%85%e7%ba%a7%e9%a9%ac%e9%87%8c%e5%a5%a5)
- [关于ShaderToy](#%e5%85%b3%e4%ba%8eshadertoy)
- [关于ShaderEffect](#%e5%85%b3%e4%ba%8eshadereffect)
- [ShaderToy原理](#shadertoy%e5%8e%9f%e7%90%86)
  - [约定的变量](#%e7%ba%a6%e5%ae%9a%e7%9a%84%e5%8f%98%e9%87%8f)
  - [glsl版本号](#glsl%e7%89%88%e6%9c%ac%e5%8f%b7)
  - [glsl版本兼容](#glsl%e7%89%88%e6%9c%ac%e5%85%bc%e5%ae%b9)
  - [ShaderToy适配](#shadertoy%e9%80%82%e9%85%8d)
- [TaoShaderToy](#taoshadertoy)

## 简介

这是《Qml特效》系列文章的特别篇，涛哥将会教大家移植ShaderToy的特效到Qml。

## 关于文章

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 效果预览

先看几个效果图

### 穿云洞
![](/images/ShaderToy/Preview1.gif)

### 星球之光
![](/images/ShaderToy/Preview2.gif)

### 蜗牛
![](/images/ShaderToy/Preview3.gif)

### 超级马里奥
![](/images/ShaderToy/Preview4.gif)

gif录制质量较低，可编译运行源码或使用涛哥打包好的可执行程序，查看实际运行效果。

源码仓库1 [https://github.com/jaredtao/TaoQuick](https://github.com/jaredtao/TaoQuick)

源码仓库2 [https://github.com/jaredtao/TaoShaderToy](https://github.com/jaredtao/TaoShaderToy)

可执行程序下载链接(包括windows 和 MacOS平台) [https://github.com/jaredtao/TaoQuick/releases](https://github.com/jaredtao/TaoQuick/releases)  

## 关于ShaderToy

学习过计算机图形学的人，都应该知道大名鼎鼎的ShaderToy网站 

用一些Shader代码和简单的纹理，就可以输出各种酷炫的图形效果和音频效果。

如果你还不知道，赶紧去看看吧[https://www.shadertoy.com](https://www.shadertoy.com)

顺便提一下，该网站的作者是IQ大神，这里有他的博客：

http://www.iquilezles.org/www/articles/raymarchingdf/raymarchingdf.htm

本文主要讨论图形效果，音频效果以后再实现。

## 关于ShaderEffect

Qml中实现ShaderToy，最快的途径就是ShaderEffect了。

上一篇文章《Qml特效-着色器效果ShaderEffect》已经介绍过ShaderEffect了, 本文重点是移植ShaderToy。

在涛哥写这篇文章之前，已经有两位前辈做过相关的研究。

陈锦明： https://zhuanlan.zhihu.com/p/38942460

qyvlik: https://zhuanlan.zhihu.com/p/44417680

涛哥参考了他们的实现，做了一些改进、完善。

在此感谢两位前辈。

下面正文开始

## ShaderToy原理

OpenGL的可编程渲染管线中，着色器代码是可以动态编译、加载到GPU运行的。

而OpenGL又包括了桌面版(OpenGL Desktop)、嵌入式版(OpenGL ES)以及网页版(WebGL)

ShaderToy网站是以WebGL 2.0为基础，提供内置函数、变量，并约定了一些输入变量，由用户按照约定编写着色器代码。

只要不是太老的OpenGL版本，内置函数、变量基本都是通用的。

### 约定的变量

ShaderToy网站约定的变量如下：

```glsl
vec3        iResolution             image/buffer        The viewport resolution (z is pixel aspect ratio, usually 1.0)
float       iTime                   image/sound/buffer	Current time in seconds
float       iTimeDelta              image/buffer        Time it takes to render a frame, in seconds
int         iFrame                  image/buffer        Current frame
float       iFrameRate              image/buffer        Number of frames rendered per second
float       iChannelTime[4]         image/buffer        Time for channel (if video or sound), in seconds
vec3        iChannelResolution[4]	image/buffer/sound	Input texture resolution for each channel
vec4        iMouse                  image/buffer        xy = current pixel coords (if LMB is down). zw = click pixel
sampler2D	iChannel{i}             image/buffer/sound	Sampler for input textures i
vec4        iDate                   image/buffer/sound	Year, month, day, time in seconds in .xyzw
float       iSampleRate             image/buffer/sound	The sound sample rate (typically 44100)
```

Qml中的相应实现

```qml
ShaderEffect {
    id: shader

    //properties for shader

    //not pass to shader
    readonly property vector3d defaultResolution: Qt.vector3d(shader.width, shader.height, shader.width / shader.height)
    function calcResolution(channel) {
        if (channel) {
            return Qt.vector3d(channel.width, channel.height, channel.width / channel.height);
        } else {
            return defaultResolution;
        }
    }
    //pass
    readonly property vector3d  iResolution: defaultResolution
    property real       iTime: 0
    property real       iTimeDelta: 100
    property int        iFrame: 10
    property real       iFrameRate
    property vector4d   iMouse;
    property var iChannel0; //only Image or ShaderEffectSource
    property var iChannel1; //only Image or ShaderEffectSource
    property var iChannel2; //only Image or ShaderEffectSource
    property var iChannel3; //only Image or ShaderEffectSource
    property var        iChannelTime: [0, 1, 2, 3]
    property var        iChannelResolution: [calcResolution(iChannel0), calcResolution(iChannel1), calcResolution(iChannel2), calcResolution(iChannel3)]
    property vector4d   iDate;
    property real       iSampleRate: 44100
    
    ...

}
```

其中时间、日期通过Timer刷新，鼠标位置用MouseArea刷新。

同时涛哥导出了hoverEnabled、running属性和restart函数，以方便Qml中控制Shader的运行。

```qml
ShaderEffect {
    id: shader
...
    //properties for Qml controller
    property alias hoverEnabled: mouse.hoverEnabled
    property bool running: true
    function restart() {
        shader.iTime = 0
        running = true
        timer1.restart()
    }

    Timer {
        id: timer1
        running: shader.running
        triggeredOnStart: true
        interval: 16
        repeat: true
        onTriggered: {
            shader.iTime += 0.016;
        }
    }
    Timer {
        running: shader.running
        interval: 1000
        onTriggered: {
            var date = new Date();
            shader.iDate.x = date.getFullYear();
            shader.iDate.y = date.getMonth();
            shader.iDate.z = date.getDay();
            shader.iDate.w = date.getSeconds()
        }
    }
    MouseArea {
        id: mouse
        anchors.fill: parent
        onPositionChanged: {
            shader.iMouse.x = mouseX
            shader.iMouse.y = mouseY
        }
        onClicked: {
            shader.iMouse.z = mouseX
            shader.iMouse.w = mouseY
        }
    }
...
}
```
### glsl版本号

GLSL Versions

|OpenGL Version	|GLSL Version|
|--|--|
|2.0|110|
|2.1|120|
|3.0|130|
|3.1|140|
|3.2|150|
|3.3|330|
|4.0|400|
|4.1|410|
|4.2|420|
|4.3|430|

GLSL ES Versions (Android, iOS, WebGL)

|OpenGL ES Version|	GLSL ES Version|
|--|--|
|2.0|100|
|3.0|300|

### glsl版本兼容

ShaderToy限定了WebGL 2.0，而我们移植到Qml中，自然是希望能够在所有可以运行Qml的设备上运行ShaderToy效果。

所以要做一些glsl版本相关的处理。

涛哥研究了Qt的GraphicsEffects模块源码，它的版本处理要么默认，要么 150 core，显然是不够用的。

glsl各个版本的差异，可以参考这里 [https://github.com/mattdesl/lwjgl-basics/wiki/glsl-versions](https://github.com/mattdesl/lwjgl-basics/wiki/glsl-versions)

涛哥总结出了如下的代码和注释说明：

注意"#version  xxx"必须是着色器的第一行，不能换行

```qml
    
    // 如果环境是OpenGL ES2，默认的version是 version 110, 不需要写出来。
    // 比ES2更老的版本是ES 1.0 和 ES 1.1, 这种古董设备，建议还是不要玩Shader了吧。
    // ES2没有texture函数，要用旧的texture2D代替
    // 精度限定要写成float

    readonly property string gles2Ver: "
#define texture texture2D
precision mediump float;
"
    // 如果环境是OpenGL ES3，version是 version 300 es
    // ES 3.1 ES 3.2也可以。
    // ES3可以用in out 关键字，gl_FragColor也可以用out fragColor取代
    // 精度限定要写成float

    readonly property string gles3Ver: "#version 300 es
#define varying in
#define gl_FragColor fragColor
precision mediump float;

out vec4 fragColor;
"
    // 如果环境是OpenGL Desktop 3.x，version这里参考Qt默认的version 150。大部分Desktop设备应该
    // 都是150, 即3.2版本，第一个区分Core和Compatibility的版本。
    // Core是核心模式，只有核心api以减轻负担。相应的Compatibility是兼容模式，保留全部API以兼容低版本。
    // Desktop 3.x 可以用in out 关键字，gl_FragColor也可以用out fragColor取代
    // 精度限定抹掉，用默认的。不抹掉有些情况下会报错，不能通用。
    readonly property string gl3Ver: "#version 150
#define varying in
#define gl_FragColor fragColor
#define lowp
#define mediump
#define highp

out vec4 fragColor;
"
    // 如果环境是OpenGL Desktop 2.x，version这里就用2.0的version 110，即2.0版本
    // 2.x 没有texture函数，要用旧的texture2D代替
    readonly property string gl2Ver: "#version 110
#define texture texture2D
"
    property string versionString: {
        if (Qt.platform.os === "android") {
            if (GraphicsInfo.majorVersion === 3) {
                console.log("android gles 3")
                return gles3Ver
            } else {
                console.log("android gles 2")
                return gles2Ver
            }
        } else {
            if (GraphicsInfo.majorVersion === 3 ||GraphicsInfo.majorVersion === 4) {
                return gl3Ver
            } else {
                return gl2Ver
            }
        }
    }
    readonly property string forwardString: versionString + "
        varying vec2 qt_TexCoord0;
        varying vec4 vertex;
        uniform lowp   float qt_Opacity;

        uniform vec3   iResolution;
        uniform float  iTime;
        uniform float  iTimeDelta;
        uniform int    iFrame;
        uniform float  iFrameRate;
        uniform float  iChannelTime[4];
        uniform vec3   iChannelResolution[4];
        uniform vec4   iMouse;
        uniform vec4    iDate;
        uniform float   iSampleRate;
        uniform sampler2D   iChannel0;
        uniform sampler2D   iChannel1;
        uniform sampler2D   iChannel2;
        uniform sampler2D   iChannel3;
    "
```
versionString 这里，主要测试了Desktop和 android设备，Desktop只要显卡不太搓，都能运行的。

Android ES3的也是全部支持，ES2的部分不能运行，比如iq大神的蜗牛Shader，使用了textureLod等一系列内置函数，就不能在ES2上面跑。

### ShaderToy适配

本来是不需要写顶点着色器的。如果我们想把ShaderToy做成一个任意坐标开始的Item来用，就需要适配一下坐标。

涛哥写的顶点着色器如下，仅在默认着色器的基础上，传递qt_Vertex给下一阶段的vertex
```qml

    vertexShader: "
              uniform mat4 qt_Matrix;
              attribute vec4 qt_Vertex;
              attribute vec2 qt_MultiTexCoord0;
              varying vec2 qt_TexCoord0;
              varying vec4 vertex;
              void main() {
                  vertex = qt_Vertex;
                  gl_Position = qt_Matrix * vertex;
                  qt_TexCoord0 = qt_MultiTexCoord0;
              }"

```
片段着色器这里处理一下，适配出一个符合shaderToy的mainImage作为入口函数

```qml
    readonly property string startCode: "
        void main(void)
        {
            mainImage(gl_FragColor, vec2(vertex.x, iResolution.y - vertex.y));
        }"
    readonly property string defaultPixelShader: "
        void mainImage(out vec4 fragColor, in vec2 fragCoord)
        {
            fragColor = vec4(fragCoord, fragCoord.x, fragCoord.y);
        }"
    property string pixelShader: ""
    fragmentShader: forwardString + (pixelShader ? pixelShader : defaultPixelShader) + startCode
```
稍微说明一下，qyvlik大佬的Shader使用gl_FragCoord作为片段坐标传进去了，这种用法的ShaderToy坐标将会占据整个Qml的窗口，

而实际ShaderToy坐标不是整个窗口的时候，超出去的地方就会被切掉，显示出来的只有一小部分。

涛哥研究了一番后，顶点着色器把vertex传过来，vertex.x就是x坐标，vertex.y坐标从上到下是0 - height，而gl_FragCoord 从下到上是0 - height，

所以要翻一下。

## TaoShaderToy

最后，看一下代码的全貌吧

```qml
//TaoShaderToy.qml
import QtQuick 2.12
import QtQuick.Controls 2.12
/*
vec3        iResolution             image/buffer        The viewport resolution (z is pixel aspect ratio, usually 1.0)
float       iTime                   image/sound/buffer	Current time in seconds
float       iTimeDelta              image/buffer        Time it takes to render a frame, in seconds
int         iFrame                  image/buffer        Current frame
float       iFrameRate              image/buffer        Number of frames rendered per second
float       iChannelTime[4]         image/buffer        Time for channel (if video or sound), in seconds
vec3        iChannelResolution[4]	image/buffer/sound	Input texture resolution for each channel
vec4        iMouse                  image/buffer        xy = current pixel coords (if LMB is down). zw = click pixel
sampler2D	iChannel{i}             image/buffer/sound	Sampler for input textures i
vec4        iDate                   image/buffer/sound	Year, month, day, time in seconds in .xyzw
float       iSampleRate             image/buffer/sound	The sound sample rate (typically 44100)
*/
ShaderEffect {
    id: shader

    //properties for shader

    //not pass to shader
    readonly property vector3d defaultResolution: Qt.vector3d(shader.width, shader.height, shader.width / shader.height)
    function calcResolution(channel) {
        if (channel) {
            return Qt.vector3d(channel.width, channel.height, channel.width / channel.height);
        } else {
            return defaultResolution;
        }
    }
    //pass
    readonly property vector3d  iResolution: defaultResolution
    property real       iTime: 0
    property real       iTimeDelta: 100
    property int        iFrame: 10
    property real       iFrameRate
    property vector4d   iMouse;
    property var iChannel0; //only Image or ShaderEffectSource
    property var iChannel1; //only Image or ShaderEffectSource
    property var iChannel2; //only Image or ShaderEffectSource
    property var iChannel3; //only Image or ShaderEffectSource
    property var        iChannelTime: [0, 1, 2, 3]
    property var        iChannelResolution: [calcResolution(iChannel0), calcResolution(iChannel1), calcResolution(iChannel2), calcResolution(iChannel3)]
    property vector4d   iDate;
    property real       iSampleRate: 44100

    //properties for Qml controller
    property alias hoverEnabled: mouse.hoverEnabled
    property bool running: true
    function restart() {
        shader.iTime = 0
        running = true
        timer1.restart()
    }
    Timer {
        id: timer1
        running: shader.running
        triggeredOnStart: true
        interval: 16
        repeat: true
        onTriggered: {
            shader.iTime += 0.016;
        }
    }
    Timer {
        running: shader.running
        interval: 1000
        onTriggered: {
            var date = new Date();
            shader.iDate.x = date.getFullYear();
            shader.iDate.y = date.getMonth();
            shader.iDate.z = date.getDay();
            shader.iDate.w = date.getSeconds()
        }
    }
    MouseArea {
        id: mouse
        anchors.fill: parent
        onPositionChanged: {
            shader.iMouse.x = mouseX
            shader.iMouse.y = mouseY
        }
        onClicked: {
            shader.iMouse.z = mouseX
            shader.iMouse.w = mouseY
        }
    }
    // 如果环境是OpenGL ES2，默认的version是 version 110, 不需要写出来。
    // 比ES2更老的版本是ES 1.0 和 ES 1.1, 这种古董设备，还是不要玩Shader了吧。
    // ES2没有texture函数，要用旧的texture2D代替
    // 精度限定要写成float
    readonly property string gles2Ver: "
#define texture texture2D
precision mediump float;
"
    // 如果环境是OpenGL ES3，version是 version 300 es
    // ES 3.1 ES 3.2也可以。
    // ES3可以用in out 关键字，gl_FragColor也可以用out fragColor取代
    // 精度限定要写成float
    readonly property string gles3Ver: "#version 300 es
#define varying in
#define gl_FragColor fragColor
precision mediump float;

out vec4 fragColor;
"
    // 如果环境是OpenGL Desktop 3.x，version这里参考Qt默认的version 150。大部分Desktop设备应该都是150
    // 150 即3.2版本，第一个区分Core和Compatibility的版本。Core是核心模式，只有核心api以减轻负担。相应的Compatibility是兼容模式，保留全部API以兼容低版本。
    // 可以用in out 关键字，gl_FragColor也可以用out fragColor取代
    // 精度限定抹掉，用默认的。不抹掉有些情况下会报错，不能通用。
    readonly property string gl3Ver: "#version 150
#define varying in
#define gl_FragColor fragColor
#define lowp
#define mediump
#define highp

out vec4 fragColor;
"
    // 如果环境是OpenGL Desktop 2.x，version这里就用2.0的version 110，即2.0版本
    // 2.x 没有texture函数，要用旧的texture2D代替
    readonly property string gl2Ver: "#version 110
#define texture texture2D
"

    property string versionString: {
        if (Qt.platform.os === "android") {
            if (GraphicsInfo.majorVersion === 3) {
                console.log("android gles 3")
                return gles3Ver
            } else {
                console.log("android gles 2")
                return gles2Ver
            }
        } else {
            if (GraphicsInfo.majorVersion === 3 ||GraphicsInfo.majorVersion === 4) {
                return gl3Ver
            } else {
                return gl2Ver
            }
        }
    }

    vertexShader: "
              uniform mat4 qt_Matrix;
              attribute vec4 qt_Vertex;
              attribute vec2 qt_MultiTexCoord0;
              varying vec2 qt_TexCoord0;
              varying vec4 vertex;
              void main() {
                  vertex = qt_Vertex;
                  gl_Position = qt_Matrix * vertex;
                  qt_TexCoord0 = qt_MultiTexCoord0;
              }"
    readonly property string forwardString: versionString + "
        varying vec2 qt_TexCoord0;
        varying vec4 vertex;
        uniform lowp   float qt_Opacity;

        uniform vec3   iResolution;
        uniform float  iTime;
        uniform float  iTimeDelta;
        uniform int    iFrame;
        uniform float  iFrameRate;
        uniform float  iChannelTime[4];
        uniform vec3   iChannelResolution[4];
        uniform vec4   iMouse;
        uniform vec4    iDate;
        uniform float   iSampleRate;
        uniform sampler2D   iChannel0;
        uniform sampler2D   iChannel1;
        uniform sampler2D   iChannel2;
        uniform sampler2D   iChannel3;
    "
    readonly property string startCode: "
        void main(void)
        {
            mainImage(gl_FragColor, vec2(vertex.x, iResolution.y - vertex.y));
        }"
    readonly property string defaultPixelShader: "
        void mainImage(out vec4 fragColor, in vec2 fragCoord)
        {
            fragColor = vec4(fragCoord, fragCoord.x, fragCoord.y);
        }"
    property string pixelShader: ""
    fragmentShader: forwardString + (pixelShader ? pixelShader : defaultPixelShader) + startCode
}

```


