---
title: 玩转Qt(1)-输出彩色log
photos: /images/ConsoleColor/Image1.png
tags:
  - Qt
  - 彩色log
categories: 玩转Qt
abbrlink: 28910
date: 2019-04-29 23:44:23
---
## 简介

我们在Qt Creator中开发程序的时候，经常要做的一件事情，就是看程序的输出Log。
一般的log信息都是黑白的，比如这样的：
![fa4bb028a52e28887c9ec2a3c8b192dc.png](/images/ConsoleColor/Image1.png)

涛哥在这里告诉大家一个隐藏的技能，那就是输出彩色的log:

![293105e6f37901923d3576fcd12c650f.png](/images/ConsoleColor/2.jpg)


从此看到的log，不再是黑白的，而是五颜六色的，生活更加绚丽多彩。

## 原理
要输出彩色信息有点类似于html的语法，即在要输出的文字前加上转义字符。
指令格式如下\033[*m
这里的*就是转义字符，例如我们要输出一段绿色的文字,则
qDebug() << "\033[32m" <<"Hello!";
即在输出文字前，先输出一个颜色指令。
注意这个指令对后续的输出都会生效，如果想关掉颜色只要再输出0号指令即可
qDebug() << "\033[0m";

这里有一个指令表

```
　　0 : Reset Color Attributes
　　1 : 加粗
　　2 : 去粗
　　4 : 下划线
　　5 : 闪烁
　　7 : 反色
　　21/22 : 加粗 正常
　　24 : 去掉下划线
　　25 : 停止闪烁
　　27 : 反色
　　30 : 前景，黑色
　　31 : 前景，红色
　　32 : 前景，绿色
　　33 : 前景，黄色
　　34 : 前景，篮色
　　35 : 前景，紫色
　　36 : 前景，青色
　　37 : 前景，白色
　　40 : 背景，黑色
　　41 : 背景，红色
　　42 : 背景，绿色
　　43 : 背景，黄色
　　44 : 背景，篮色
　　45 : 背景，紫色
　　46 : 背景，青色
　　47 : 背景，白色
其它转义字符命令
    清除屏幕 : /033c
　　设定水平标位置 : /033[XG
　　X为水平标位置。
　　设定垂直标位置 : /033[Xd
　　Y为垂直标位置。
    /033[0K : 删除从标到该行结尾
　　/033[1K : 删除从该行开始到标处
　　/033[2K : 删除整行　
　　/033[0J : 删除标到萤幕结尾
　　/033[1J : 删除从萤幕开始到标处
　　/033[2J : 删除整个屏幕

```
涛哥在QtCreator中测试了，只有颜色和加粗指令能生效。
## 代码
（代码使用C++11标准）

为了方便、直观地使用，涛哥定义了一套枚举
```c++
enum class LogType {
    Reset = 0,

    Bold,
    Unbold,

    FrontBlack,
    FrontRed,
    FrontGreen,
    FrontYellow,
    FrontBlue,
    FrontPurple,
    FrontCyan,
    FrontWhite,
    BackBlack,
    BackRed,
    BackGreen,
    BackYellow,
    BackBlue,
    BackPurple,
    BackCyan,
    BackWhite,

    TypeCount
};
```
之后又写了一个指令列表，顺序和前面的枚举一一对应
```c++
static const char * logCommands[] = {
    "\033[0m",
    "\033[1m",
    "\033[2m",
    "\033[30m",
    "\033[31m",
    "\033[32m",
    "\033[33m",
    "\033[34m",
    "\033[35m",
    "\033[36m",
    "\033[37m",
    "\033[40m",
    "\033[41m",
    "\033[42m",
    "\033[43m",
    "\033[44m",
    "\033[45m",
    "\033[46m",
    "\033[47m",
};
```
这样就可以通过查表的方式，拿到对应的颜色指令了
```c++
    logCommands[(int)(LogType::FrontYellow)]
```
这个代码使用了c的强制转换，在c++11标准中会报警告，建议用static_cast。
所以涛哥又写了一个转换函数,将枚举通过static_cast转换成int类型。

```cpp
template <typename EnumType, typename IntType = int>
int enumToInt(EnumType enumValue)
{
    static_assert (std::is_enum<EnumType>::value, "EnumType must be enum");

    return static_cast<IntType>(enumValue);
}
```


最后就是代码的全貌了
```c++
#include <QDebug>
enum class LogType {
    Reset = 0,

    Bold,
    Unbold,

    FrontBlack,
    FrontRed,
    FrontGreen,
    FrontYellow,
    FrontBlue,
    FrontPurple,
    FrontCyan,
    FrontWhite,
    BackBlack,
    BackRed,
    BackGreen,
    BackYellow,
    BackBlue,
    BackPurple,
    BackCyan,
    BackWhite,

    TypeCount
};
static const char * logCommands[] = {
    "\033[0m",
    "\033[1m",
    "\033[2m",
    "\033[30m",
    "\033[31m",
    "\033[32m",
    "\033[33m",
    "\033[34m",
    "\033[35m",
    "\033[36m",
    "\033[37m",
    "\033[40m",
    "\033[41m",
    "\033[42m",
    "\033[43m",
    "\033[44m",
    "\033[45m",
    "\033[46m",
    "\033[47m",
};
template <typename EnumType, typename IntType = int>
int enumToInt(EnumType enumValue)
{
    static_assert (std::is_enum<EnumType>::value, "EnumType must be enum");

    return static_cast<IntType>(enumValue);
}
int main(int argc, char *argv[])
{
    for (int i = enumToInt(LogType::Bold); i < enumToInt(LogType::TypeCount); ++i)
    {
        qInfo().nospace() << logCommands[i] << i << " Hello World" << logCommands[0];
    }
    qWarning() << logCommands[enumToInt(LogType::FrontBlue)]
               << logCommands[enumToInt(LogType::BackRed)]
               << u8"感谢大家对涛哥系列文章的支持，也"
                  "欢迎直接联系我寻求帮助" << logCommands[0];
    return 0;
}

```
顺便说一下，设置了console的工程不能显示出彩色，得把console去掉。
![a03d569415d1c9ab1aaa6fb1a1d62d3a.png](/images/ConsoleColor/3.jpg)