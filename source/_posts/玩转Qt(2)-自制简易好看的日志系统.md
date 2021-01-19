---
title: 玩转Qt(2)-自制简易好看的日志系统
photos: /images/Logger/1.png
tags:
  - Qt
  - Log
  - 日志系统
  - 轻量级Log系统
categories: 玩转Qt
abbrlink: 60472
date: 2019-04-30 23:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [预览](#%E9%A2%84%E8%A7%88)
- [原理](#%E5%8E%9F%E7%90%86)
  - [html格式的log](#html%E6%A0%BC%E5%BC%8F%E7%9A%84log)
  - [Qt的log系统](#qt%E7%9A%84log%E7%B3%BB%E7%BB%9F)
  - [融合](#%E8%9E%8D%E5%90%88)
- [文件句柄复用](#%E6%96%87%E4%BB%B6%E5%8F%A5%E6%9F%84%E5%A4%8D%E7%94%A8)
- [多线程测试](#%E5%A4%9A%E7%BA%BF%E7%A8%8B%E6%B5%8B%E8%AF%95)
- [github仓库链接](#github%E4%BB%93%E5%BA%93%E9%93%BE%E6%8E%A5)

## 简介

一个完善的软件工程，自然是少不了log系统的。

这次涛哥教大家，用最少的代码做一个轻量又好看的log系统。

涛哥知道有现成的log4cpp、log4cplus之类的，也有使用过。

这次是抱着学习的心态来造这个轮子的，造轮子的过程才能学到

更多知识，才能有进步、有提升，难道不是么？

## 预览

先看一下成果

![预览](/images/Logger/1.png)

## 原理

### html格式的log

为了实现 “代码最少” 和 “好看” 的需求，涛哥把log写进了一个html文件。

这样的log相当于一个静态的网页，只要装有浏览器的操作系统，都可以打开并看到上面图示那样的log。

涛哥给这个html文件设计了一个固定的模板的:

```html
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html>

<head>
    <title>TaoLogger</title>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <style type="text/css" id="logCss">
        body {
            background: #18242b;
            color: #afc6d1;
            margin-right: 20px;
            margin-left: 20px;
            font-size: 14px;
            font-family: Arial, sans-serif, sans;
        }

        a {
            text-decoration: none;
        }

        a:link {
            color: #a0b2bb;
        }

        a:active {
            color: #f59504;
        }

        a:visited {
            color: #adc7d4;
        }

        a:hover {
            color: #e49115;
        }

        h1 {
            text-align: center;
        }

        h2 {
            color: #ebe5e5;
        }

        .d,
        .w,
        .c,
        .f,
        .i {
            padding: 3px;
            overflow: auto;
        }

        .d {
            background-color: #0f1011;
            color: #a8c1ce;
        }

        .i {
            background-color: #294453;
            color: #a8c1ce;
        }

        .w {
            background-color: #7993a0;
            color: #1b2329;
        }

        .c {
            background-color: #ff952b;
            color: #1d2930;
        }

        .f {
            background-color: #fc0808;
            color: #19242b;
        }
    </style>
</head>

<body>
    <h1><a href="https://jaredtao.github.io">TaoLogger</a> 日志文件</h1>
    <script type="text/JavaScript">
        function objHide(obj) {
            obj.style.display="none"
        }
        function objShow(obj) {
            obj.style.display="block"
        }
        function selectType() {
            var sel = document.getElementById("typeSelect");
            const hideList = new Set(['d', 'i', 'w', 'c', 'f']);
            if (sel.value === 'a') {
                hideList.forEach(element => {
                    var list = document.querySelectorAll('.' + element);
                    list.forEach(objShow);
                });
            } else {
                var ss = hideList;
                ss.delete(sel.value);
                ss.forEach(element => {
                    var list = document.querySelectorAll('.' + element);
                    list.forEach(objHide);
                });
                var showList = document.querySelectorAll('.' + sel.value);
                showList.forEach(objShow);
            }
        }
    </script>
    <select id="typeSelect" onchange="selectType()">
        <option value='a' selected="selected">All</option>
        <option value='d'>Debug</option>
        <option value='i'>Info</option>
        <option value='w'>Warning</option>
        <option value='c'>Critical</option>
        <option value='f'>Fatal</option>
    </select>
```
（如果你不懂html，也没关系，直接拿过去用就好了）

这个模板只使用了一些很基本的html元素和css样式表，筛选器那里用了一点JavaScript。

* Log模板的用法

很简单的，模板作为html文件的前面部分，接下来每一行log，以追加的方式跟在模板后面就行了。

(html的body结束标记并没有写，浏览器都能正常打开。容错性真的强！)

当然, 每一条log有个格式要求:

```html
    <div class="d"> 山有木兮木有枝，心悦君兮君不知。</div>
```

就是增加了一对div标记， div的class属性要设置为d、i、w、c、f这几个字符中的一个，分别是

debug、info、warning、critical、fatal的首字母, 这正是Qt所提供的log分类。

设置div的class属性，就是给筛选器用来做筛选。

* Log模板的存取

文件读取? 不，太慢了。

这就是一段固定的字符串，直接编译进代码里，程序启动的时候直接装载到内存就好了。

那么C++里面，怎么才能装下这段带有转义字符的字符串呢？涛哥的答案是：C++11的 “原始字符串字面量”或者叫 “R字符串”

可以参考这里 [cppreference](https://zh.cppreference.com/w/cpp/language/string_literal)

简单来说，是这样写的：

```c++
string logTemplate = R"(xxxxxx)";
```

只要有了 R"(  )" 这个写法，括号中间随便写转义字符、换行符都行。当然为了方便让编译器识别哪个

才是真正的'结束括号'，C++11标准提出了括号前后增加分隔符的写法，即:
```c++
string logTemplate = R"prefix(xxxxxx)prefix";
```
左括号的前面和右括号的后面, 是同样的一段字符串作为分隔符就行了。

涛哥的代码里是这么用的
```c++
namespace Logger
{
    const static QString logTemplate = u8R"logTemplate(
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
<html>

<head>
    <title>TaoLogger</title>
    ...
    这里省略一大堆html代码
    ...

    )logTemplate";

}
```
### Qt的log系统

* Qt的log分类

Qt的打印信息，大家普遍使用的是qDebug，不过Qt除了qDebug，还有qInfo, qWarning, qCritical等等。

涛哥翻了Qt5.12的源码，发现这几个打印最终都是通过fprintf(stderr)或者fprintf(stdout)来实现输出的，

不同的地方就在于Log类型。如果要用好这个分类，那我们平时使用打印的时候，就要注意做区分:


    - 调试信息用qDebug

    - 常规信息用qInfo

    - 警告用qWarning

    - 比较严重的问题用qCritical


* Qt的log格式化

Qt提供了一个函数qSetMessagePattern，用来定制输出信息。

例如：
```c++
    qSetMessagePattern("[%{time yyyyMMdd h:mm:ss.zzz t} %{if-debug}D%{endif}%{if-info}I%{endif}%{if-warning}W%{endif}%{if-critical}C%{endif}%{if-fatal}F%{endif}] %{file}:%{line} - %{message}");
```
一般只要在main.cpp中添加这一行代码，之后的qDebug、qInfo等函数都会按照这个格式来输出，包含了

时间戳、log类型、文件名、行号 等信息。也可以不改任何代码、改环境变量来做到

![预览](/images/Logger/2.jpg) 
![预览](/images/Logger/3.jpg)

* Release模式信息缺失


这里有个问题，就是文件名和行号在debug模式正常，Release模式会变成空的。

要解决这个问题，那么就需要编译器提供的内置宏`__FILE__` 和 `__LINE__`了

涛哥写了这样几个宏，代替qDebug和qInfo等函数。
```C++
#define LOG_DEBUG qDebug() << __FILE__ << __FUNCTION__ << __LINE__
#define LOG_INFO qInfo() << __FILE__ << __FUNCTION__ << __LINE__
#define LOG_WARN qWarning() << __FILE__ << __FUNCTION__ << __LINE__
#define LOG_CRIT qCritical() << __FILE__ << __FUNCTION__ << __LINE__

```

用法类似这样:

```C++
    LOG_DEBUG << u8"山有木兮木有枝，心悦君兮君不知。";
```


* Qt的写log文件

Qt还提供了一个函数 qInstallMessageHandler，可以插入一个回调函数，让每一行qDebug/qInfo等

函数的打印信息，都经过这个回调来处理。看一下帮助文档：

![预览](/images/Logger/4.jpg)

其实帮助文档已经提供了一个简易的log功能，涛哥就是在这个功能的基础上，做了一些定制化的修改。

### 融合
  
![预览](/images/Logger/5.jpg)

* log存储路径和容量

涛哥写了一个函数和一组静态变量，用来设置和记录log存储的路径和容量

头文件中的声明
```c++
#pragma once
#include <QDebug>

namespace Logger
{
//默认存储路径为当前路径的Log文件夹下，默认文件数量为1024
void initLog(const QString& logPath = QStringLiteral("Log"), int logMaxCount = 1024);

} // namespace Logger

```
CPP中的实现
```C++
namespace Logger
{
//静态变量，记录存储路径
static QString gLogDir;
//静态变量，记录最大存储数量
static int gLogMaxCount;

void initLog(const QString &logPath, int logMaxCount)
{
    //安装回调
    qInstallMessageHandler(outputMessage);
    //记录路径
    gLogDir = QCoreApplication::applicationDirPath() + "/" + logPath;
    //记录最大存储数
    gLogMaxCount = logMaxCount;
    //检查存储文件夹，不存在则创建
    QDir dir(gLogDir);
    if (!dir.exists())
    {
        dir.mkpath(dir.absolutePath());
    }
    //获取文件列表
    QStringList infoList = dir.entryList(QDir::Files, QDir::Name);
    //硬盘空间有限，超过最大存储数的都删掉。
    while (infoList.size() > gLogMaxCount)
    {
        //每次删第一个。文件名其实是默认按时间排序的，第一个就是时间最早的。
        dir.remove(infoList.first());
        infoList.removeFirst();
    }
}
static void outputMessage(QtMsgType type, const QMessageLogContext &context, const QString &msg)
{
    //
}
}
```

* log存储

```C++
static void outputMessage(QtMsgType type, const QMessageLogContext &context, const QString &msg)
{
    //每一条消息的约定格式。%1即log类型，%2即log内容。这里用静态变量，每次用的时候填充
    //生成一个QString副本，达到最大程度的复用。
    static const QString messageTemp= QString("<div class=\"%1\">%2</div>\r\n");
    //预定的消息类型映射表
    static const char typeList[] = {'d', 'w', 'c', 'f', 'i'};
    //锁
    static QMutex mutex;
    //取时间
    QDateTime dt = QDateTime::currentDateTime();
    
    //时间作为文件名

    //每分钟一个文件
    //QString fileNameDt = dt.toString("yyyy-MM-dd_hh_mm");

    //每小时一个文件
    QString fileNameDt = dt.toString("yyyy-MM-dd_hh");

    //每天一个文件
    //QString fileNameDt = dt.toString("yyyy-MM-dd_");
    //时间戳
    QString contentDt = dt.toString("yyyy-MM-dd hh:mm:ss");
    //消息的前面写上时间戳，后面写内容。 msg如果是用LOG_WARN那几个宏打印的，本身已经带了文件名和行号了。
    QString message = QString("%1 %2").arg(contentDt).arg(msg);
    
    //组装一条html格式的log
    QString htmlMessage = messageTemp.arg(typeList[static_cast<int>(type)]).arg(message);

    QFile file(QString("%1/%2_log.html").arg(gLogDir).arg(fileNameDt));
    //这里开始锁起来，多线程安全
    mutex.lock();
    bool exist = file.exists();
    //写 | 追加的方式
    file.open(QIODevice::WriteOnly | QIODevice::Append);
    //文件流
    QTextStream text_stream(&file);
    //注意字符编码
    text_stream.setCodec("UTF-8");
    if (!exist)
    {
        //文件不存在的情况下，先把我们的html模板写进去。
        text_stream << logTemplate << "\r\n";
    }
    //往文件流里面追加数据
    text_stream << htmlMessage;

    file.close();
    mutex.unlock();
    //解锁
    
    //把log都写到文件了，QtCreator 或者VS 不就看不到输出了？
    //这里用Win32的方式多加了一次输出，当然也可以使用std::cout fprintf。不能再使用qDebug了，因为这是在qDebug的回调里，会无限递归调用的。
    ::OutputDebugString(message.toStdWString().data());
    ::OutputDebugString(L"\r\n");
}
```



## github仓库链接

源代码代码去github吧。

[TaoLogger](https://github.com/jaredtao/TaoLogger)