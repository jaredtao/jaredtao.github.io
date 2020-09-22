---
title: Qt实用技能3-理解事件循环
photos: /img/avatar.jpg
tags:
  - Qt
  - Qt实用技能
categories: Qt进阶之路
abbrlink: 24511
date: 2019-07-06 18:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [事件与事件循环](#%E4%BA%8B%E4%BB%B6%E4%B8%8E%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF)
  - [Hello World](#Hello-World)
  - [循环处理](#%E5%BE%AA%E7%8E%AF%E5%A4%84%E7%90%86)
  - [类比事件循环的概念](#%E7%B1%BB%E6%AF%94%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF%E7%9A%84%E6%A6%82%E5%BF%B5)
- [不同操作系统的事件循环](#%E4%B8%8D%E5%90%8C%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF)
  - [Windows](#Windows)
  - [Linux X11窗口](#Linux-X11%E7%AA%97%E5%8F%A3)
  - [MacOS Cocoa Application](#MacOS-Cocoa-Application)
- [Qt的事件循环](#Qt%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF)
  - [QEventLoop类](#QEventLoop%E7%B1%BB)
  - [QCoreApplication 主事件循环](#QCoreApplication-%E4%B8%BB%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF)
- [Qt的事件分发和事件处理](#Qt%E7%9A%84%E4%BA%8B%E4%BB%B6%E5%88%86%E5%8F%91%E5%92%8C%E4%BA%8B%E4%BB%B6%E5%A4%84%E7%90%86)
  - [重载事件](#%E9%87%8D%E8%BD%BD%E4%BA%8B%E4%BB%B6)
  - [QEvent](#QEvent)
  - [事件过滤器](#%E4%BA%8B%E4%BB%B6%E8%BF%87%E6%BB%A4%E5%99%A8)
- [事件循环的运用](#%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF%E7%9A%84%E8%BF%90%E7%94%A8)
  - [processEvents不阻塞UI](#processEvents%E4%B8%8D%E9%98%BB%E5%A1%9EUI)
  - [QEventLoop模拟同步调用](#QEventLoop%E6%A8%A1%E6%8B%9F%E5%90%8C%E6%AD%A5%E8%B0%83%E7%94%A8)


## 简介

本文是《Qt实用技能》系列文章的第三篇，涛哥在这里讨论事件循环相关的知识点。

一些常见的问题，诸如：
    
    为什么Qt程序的main函数都有一个QApplication?

    执行比较耗时的任务时，怎样能不卡界面?

    同步、异步、阻塞、非阻塞到底是怎么回事?

等等，都可以在本文中找到答案。

注：文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 事件与事件循环

### Hello World

从Hello World说起吧

```C++
#include <stdio.h>
int main(int argc, char *argv[]) 
{
    printf("Hello World");
    return 0;
}
```
这是一段大家都很熟悉的命令行程序，运行起来会在终端输出"Hello World"，之后程序就退出了。

### 循环处理

我们稍微加点需求: 程序能够一直运行，每次用户输入一些信息并按下回车时，打印出用户的输入。直到输入的内容为“quit”时才退出。

按照这个需求，代码实现如下：

```C++
#include <stdio.h>
#include <string.h>
int main(int argc, char* argv[])
{
    char input[1024];   //假设输入长度不超过1024
    const char quitStr[] = "quit";
    bool quit = false;
    while (false == quit) {
        scanf_s("%s", input, sizeof input);
        printf("user input: %s\n", input);
        if (0 == memcmp(input, quitStr, sizeof quitStr)) {
            quit = true;
        }
    }
    return 0;
}
```
我们使用了一个while循环。在这个循环体内，不停地处理用户的输入。当输入的内容为"quit"时，循环终止条件被设置为true，循环将终止。

### 类比事件循环的概念

在上面这个例子中，“用户输入并按下回车”这件事情，我们可以称作一个“事件”或者“用户输入事件”，不停的去处理“事件”的这段代码，

我们可以称作“事件循环”, 也可以叫做"消息循环"，是一回事。

一般对于带UI窗口的程序来说，“事件”是由操作系统或程序框架在不同的时刻发出的。

当用户按下鼠标、敲下键盘，或者是窗口需要重新绘制的时候，计时器触发的时候，都会发出一个相应的事件。

我们把“事件循环”的代码 提炼/抽象 如下：

```C++
function loop() {
    initialize();
    bool shouldQuit = false;
    while(false == shouldQuit)
    {
        var message = get_next_message();
        process_message(message);
        if (message == QUIT) 
        {
            shouldQuit = true;
        }
    }
}
```
在事件循环中, 不停地去获取下一个事件，然后做出处理。直到quit事件发生，循环结束。


有“取事件”的过程，那么自然有“存储事件”的地方，要么是操作系统存储，要么是软件框架存储。

存储事件的地方，我们称作 “事件队列” Event Queue

处理事件，我们也称作 “事件分发” Event Dispatch


## 不同操作系统的事件循环

### Windows

先来看一个Windows系统的事件循环示例(win32 API)：

```C++
    MSG msg = { 0 };
    bool done = false;
    bool result = false;
    while (!done)
    {
        if (PeekMessage(&msg, 0, 0, 0, PM_REMOVE))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
        }
        if (msg.message == WM_QUIT)
        {
            done = true;
        }
    }
```

思路和前面介绍的一致

### Linux X11窗口

有些linux系统使用X11窗口系统，看看其窗口事件循环

```C++
Atom wmDeleteMessage = XInternAtom(mDisplay, "WM_DELETE_WINDOW", False);
XSetWMProtocols(display, window, &wmDeleteMessage, 1);

XEvent event;
bool running = true;

while (running)
{
    XNextEvent(display, &event);

    switch (event.type)
    {
        case Expose:
            printf("Expose\n");
            break;

        case ClientMessage:
            if (event.xclient.data.l[0] == wmDeleteMessage)
                running = false;
            break;

        default:
            break;
    }
}

```
思路也是和前面一致的

### MacOS Cocoa Application

在Cocoa Application中, 有一种获取事件的机制，叫做runloop(一个NSRunLoop对象,它允许进程接收窗口服务的各种事件)

一般的Cocoa Application运行流程是，从runloop的事件队列中获取一个事件(NSEvent)

派发事件(NSEvent)到合适的对象(Object)

事件被处理完成后,再取下一个事件(NSEvent),直到应用退出.

思路也是和前面一致的。

## Qt的事件循环

Qt作为一个跨平台的UI框架，其事件循环实现原理, 就是把不同平台的事件循环进行了封装，并提供统一的抽象接口。

和Qt做了类似工作的，还有glfw、SDL等等很多开源库。

### QEventLoop类

QEventLoop即Qt中的事件循环类，主要接口如下：
```C++
int exec(QEventLoop::ProcessEventsFlags flags = AllEvents)
void exit(int returnCode = 0)
bool isRunning() const
bool processEvents(QEventLoop::ProcessEventsFlags flags = AllEvents)
void processEvents(QEventLoop::ProcessEventsFlags flags, int maxTime)
void wakeUp()
```
其中exec是启动事件循环，调用exec以后，调用exec的函数就会被“阻塞”，直到EventLoop里面的while循环结束。

这里画个简单的示意图:

![预览](/images/Qt3/1.png)

exit是退出事件循环(将EventLoop中的退出标识设为true)

processEvents是及时处理队列中的事件(这个很有用，后面还会讲)。

这里有个问题，exec阻塞了当前函数，还怎么退出EventLoop呢？

答案是：在派发事件后，某个事件处理的函数中，达到事件退出条件时，调用exit函数，将EventLoop中的退出标识设为true。

![预览](/images/Qt3/2.png)

这样的程序运行流程，我们叫做 “事件驱动”式的程序。

### QCoreApplication 主事件循环

一般的Qt程序，main函数中都有一个QCoreApplication/QGuiApplication/QApplication，并在末尾调用 exec。
```C++
int main(int argc, char *argv[])
{
    QCoreApplication app(argc, argv);
    //或者QGuiApplication， 或者 QApplication
    ...
    ...
    return app.exec();
}
```
Application类中，除去启动参数、版本等相关东西后，关键就是维护了一个QEventLoop，Application的exec就是QEventLoop的exec。

不过Application中的这个EventLoop，我们称作“主事件循环”Main EventLoop。

所有的事件分发、事件处理都从这里开始。

Application还提供了sendEvent和poseEvent两个函数，分别用来发送事件。

sendEvent发出的事件会立即被处理，也就是“同步”执行。

postEvent发送的事件会被加入事件队列，在下一轮事件循环时才处理，也就是“异步”执行。

还有一个特殊的sendPostedEvents，是将已经加入队列中的准备异步执行的事件立即同步执行。

## Qt的事件分发和事件处理

以QWidget为例来说明。

QWidget是Widget框架中，大部分UI组件的基类。QWidget类拥有一些名字为xxxEvent的虚函数,比如：

```
virtual void keyPressEvent(QKeyEvent *event)
virtual void keyReleaseEvent(QKeyEvent *event)
```
keyPressEvent就表示按键按下时的处理，keyReleaseEvent表示按键松开时的处理。

主事件循环中(注册过QWidget类之后)，事件分发会在按键按下时调用QWidget的keyPressEvent函数，按键松开时调用QWidget的keyReleaseEvent函数。

### 重载事件

有了上面的事件处理机制，我们就可以在自己的QWidget子类中，通过重载keyPressEvent、keyReleaseEvent等等事件处理函数，做一些自定义的事件处理。

### QEvent

每一个事件处理函数，都是带有参数的，这个参数是QEvent的子类，携带了各种事件的参数。比如

按键事件 void keyPressEvent(QKeyEvent *event) 中的QKeyEvent, 就包括了按下的按键值key、 count等等。

### 事件过滤器

Qt还提供了事件过滤机制，在事件分发之前先过滤一部分事件。

用法如下：
```
  class KeyPressEater : public QObject
  {
      Q_OBJECT
      ...

  protected:
      bool eventFilter(QObject *obj, QEvent *event) override;
  };

  bool KeyPressEater::eventFilter(QObject *obj, QEvent *event)
  {
      if (event->type() == QEvent::KeyPress) {
          QKeyEvent *keyEvent = static_cast<QKeyEvent *>(event);
          qDebug("Ate key press %d", keyEvent->key());
          return true;
      } else {
          // standard event processing
          return QObject::eventFilter(obj, event);
      }
  }

  。。。

   monitoredObj->installEventFilter(filterObj);
```
自定义一个QObject子类，重载eventFilter函数。之后在要过滤的QObject对象上，调用installEventFilter函数以安装过滤器上去。

过滤器函数的返回值为bool，true表示这个事件被过滤掉了，不用再往下分发了。false表示没有过滤。

## 事件循环的运用

### processEvents不阻塞UI

我们的UI界面，要持续不断地刷新（对于QWidget就是触发paintEvent事件），以保证显示流畅、能及时响应用户输入。

一般要有一个良好的帧率，比如每秒刷新60帧, 即经常说的FPS 60， 换算一下 1000 ms/ 60 ≈ 16 ms,也就是每隔16毫秒刷新一次。

而我们有时候又需要做一些复杂的计算，这些计算的耗时远远超过了16毫秒。

在没有计算完成之前，函数不会退出（相当于阻塞），事件循环得不到及时处理，就会发生UI卡住的现象。

这种场景下，就可以使用Qt为我们提供的接口，立即处理一次事件循环，来保证UI的流畅

(先不讨论多线程的情况，后续有多线程专题文章，而且你们玩的不一定比涛哥6)

```c++
//耗时操作
someWork1()
//适当的位置，插入一个processEvents,保证事件循环被处理
QCoreApplication::processEvents();

//耗时操作
someWork2()

```

### QEventLoop模拟同步调用

经常会有这种场景： “触发 ”了某项操作，必须等该操作完成后才能进行“ 下一步 ”

比如：软件的登录界面，向服务器发起登录请求后，必须等收到服务器返回的登录数据，才知道登录结果并决定下一步如何执行。

这种场景，如果设计成异步调用，直接用Qt的信号/槽即可，如果要设计成同步调用，就可以使用本地QEventLoop

这里写段伪代码示例一下：

```C++
bool login(const QString &userName, const QString &passwdHash, const QString &slat)
{
    //声明本地EventLoop
    QEventLoop loop;
    bool result = false;
    //先连接好信号
    connect(&network, &Network::result, [&](bool r, const QString &info){
        result = r;
        qDebug() << info;
        //槽中退出事件循环
        loop.quit();
    });
    //发起登录请求
    sendLoginRequest(userName, passwdHash, slat);
    //启动事件循环。阻塞当前函数调用，但是事件循环还能运行。
    //这里不会再往下运行，直到前面的槽中，调用loop.quit之后，才会继续往下走
    loop.exec();
    //返回result。loop退出之前，result中的值已经被更新了。
    return result;
}
```