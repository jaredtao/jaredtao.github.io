---
title: Qml组件化编程4-i18n动态国际化
photos: /img/avatar.jpg
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: Qml组件化编程
abbrlink: 18621
date: 2019-05-12 23:44:23
---

- [简介](#%E7%AE%80%E4%BB%8B)
- [效果预览](#%E6%95%88%E6%9E%9C%E9%A2%84%E8%A7%88)
- [源码中输出中文](#%E6%BA%90%E7%A0%81%E4%B8%AD%E8%BE%93%E5%87%BA%E4%B8%AD%E6%96%87)
- [Qt本身的国际化](#qt%E6%9C%AC%E8%BA%AB%E7%9A%84%E5%9B%BD%E9%99%85%E5%8C%96)
- [翻译工作](#%E7%BF%BB%E8%AF%91%E5%B7%A5%E4%BD%9C)
- [实现动态翻译](#%E5%AE%9E%E7%8E%B0%E5%8A%A8%E6%80%81%E7%BF%BB%E8%AF%91)
  - [加载翻译文件](#%E5%8A%A0%E8%BD%BD%E7%BF%BB%E8%AF%91%E6%96%87%E4%BB%B6)
  - [Qml中切换语言](#qml%E4%B8%AD%E5%88%87%E6%8D%A2%E8%AF%AD%E8%A8%80)
- [多国语言版本](#%E5%A4%9A%E5%9B%BD%E8%AF%AD%E8%A8%80%E7%89%88%E6%9C%AC)

## 简介

本文是《Qml组件化编程》系列文章的第四篇，涛哥将教大家，如何在Qml中实现动态国际化。

i18n 是 internationalization(国际化) 的首尾字符加中间的 18 个字符。随着产品越做越大，要推向国际的时候，国际化这一步

是必不可少的。i18n 的方案有很多，这里只讨论在Qt/Qml中的方案。

文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [知乎专栏-涛哥的Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)。

## 效果预览

看一下最终效果

![预览](/images/Qml4/1.gif)

目前支持的语言包括：
* 中文
* 英文
* 日文
* 韩文
* 法文
* 俄文
* 德文
* 西班牙文
* 葡萄牙文
* 意大利文
* 越南文
* 阿拉伯文

(其中阿拉伯文是从右往左写的，qml默认的处理看起来不完全正确。

涛哥对阿拉伯也不是很熟，就不深入研究了。)

## 源码中输出中文

先说一下源码中输出中文的问题。

在文件本身的编码是utf-8的前提下，以下三种方式都可以直接输出中文。

```
    qInfo() << u8"山有木兮木有枝，心悦君兮君不知。";
```
```
    qInfo() << QStringLiteral("黄河远上白云间，一片孤城万仞山。");
```
```
    qInfo() << QString::fromLocal8Bit("人生若只如初见，何事秋风悲画扇。");
```

u8是c++11标准支持的字符串字面量写法，可以参考https://zh.cppreference.com/w/cpp/language/string_literal

QStringLiteral是Qt特有的宏，用来在编译期生成字符串字面量，效果和u8类似。

QString::fromLocal8Bit可以在运行过程中，动态处理中文字符串。

        
## Qt本身的国际化

Qt的国际化, 相关文章太多了，Qt帮助文档也有，涛哥这里列出重点

1. C++代码中的字符串使用QObject::tr()包起来，类本身是QObject的子类时可以省略作用域“QObject::”,直接写tr
2. qml代码中使用qsTr把字符串包起来
3. pro文件中添加一句TRANSLATIONS += trans_zh.qs ，这个名字起什么无所谓，关键是‘_zh’要有。
4. 调用lrelease工具,扫描项目并生成trans_zh.qs 文件。这个文件是xml格式的，未经过翻译的，需要为这个文件做一些翻译工作。
   后面说怎么做翻译。
5. 翻译做好后，调用lupdate工具，生成trans_zh.qm文件。这个文件就是把xml变成了二进制。
6. 将qm文件放在运行路径，或者资源文件里。
7. 切换语言时， Qt/C++代码中使用QTranslater加载qm文件，QCoreApplication卸载旧的QTranslater，并安装新的QTranslater。调用
   QmlEngine::retranslate函数

在5.10以前的版本，Qt是不能直接动态切换语言的，要么重新启动程序，要么把所有的text都set一遍，retranslate是5.10才有的接口。

涛哥这次的方案，以retranslate为主，5.10以前还有各路大神的动态切换语言方案，不在本次文章的讨论范围。

## 翻译工作

Qt提供了可视化的工具，即QTDIR/bin路径下的linguist.exe，可以直接打开我们拿到的trans_zh.qs文件。这个工具可以由程序员之外不懂xml的专业翻译人士使用。

涛哥在这里就使用自动化的翻译工具来做这个事情，比如使用 百度翻译API 或 有道翻译API。

为此，涛哥使用go语言写了一个小工具，能接入百度和有道的翻译API，并且能读取Qt的qs文件、完成翻译后再写回qs文件。

小工具已经开源 [Qt-Transer](https://github.com/jaredtao/Transer)

当然这种小工具Qt也能做，涛哥使用go纯属兴趣。

## 实现动态翻译

### 加载翻译文件

需要在Qt/C++代码中实现，并提供给qml调用。这里以中文和英文为例，涛哥直接写在了自定义QQuivkView的代码里。

```c++
//TaoView.h

class TaoView : public QQuickView
{
    Q_OBJECT
public:
    explicit TaoView(QWindow *parent = nullptr);
    Q_INVOKABLE void reTrans(const QString &lang);  //这是通过invokable导出的函数

public slots:
private:
    QTranslator m_enTrans;      //英文的翻译
    QTranslator m_zhTrans;      //中文的翻译
    QString m_lang;             //记录当前语言
};

```

```C++
// TaoView.cpp
#include "stdafx.h"

#include "TaoView.h"

#include <QTranslator>
#include <QQmlEngine>
TaoView::TaoView(QWindow *parent) : QQuickView(parent)
{
    setFlag(Qt::FramelessWindowHint);
    setResizeMode(SizeRootObjectToView);
    setColor(QColor(Qt::transparent));
    
    //构造函数直接加载翻译文件

    bool ok = m_enTrans.load("trans_en.qm");
    bool ok2 = m_zhTrans.load("trans_zh.qm");
    qWarning() << ok << ok2;

    //默认安装中文
    QCoreApplication::installTranslator(&m_zhTrans);
    m_lang = QStringLiteral("中文简体");
}

void TaoView::reTrans(const QString &lang)
{
    if (m_lang == lang)
    {
        return;
    }
    //切换语言
    m_lang = lang;
    if ( lang == QStringLiteral("中文简体"))
    {
        QCoreApplication::removeTranslator(&m_enTrans);
        QCoreApplication::installTranslator(&m_zhTrans);
        engine()->retranslate();
    } else if (lang == "English") {
        QCoreApplication::removeTranslator(&m_zhTrans);
        QCoreApplication::installTranslator(&m_enTrans);
        engine()->retranslate();
    }
}

```

### Qml中切换语言

和文章3 [Qml组件化编程3-动态切换皮肤](https://jaredtao.github.io/2019/05/12/Qml%E7%BB%84%E4%BB%B6%E5%8C%96%E7%BC%96%E7%A8%8B3-%E5%8A%A8%E6%80%81%E5%88%87%E6%8D%A2%E7%9A%AE%E8%82%A4/) 类似, 在标题栏按钮中做一个切换按钮

```qml
// TitleBar.qml
TImageBtn {
    width: 20
    height: 20
    anchors.verticalCenter: parent.verticalCenter
    imageUrl: containsMouse ? "qrc:/Image/Window/lang_white.png" : "qrc:/Image/Window/lang_gray.png"
    onClicked: {
        pop.show()
    }
    TPopup {
        id: pop
        barColor: gConfig.reserverColor
        backgroundWidth: 100
        backgroundHeight: 80
        contentItem: ListView {
            id: langListView
            anchors.fill: parent
            anchors.margins: 2
            model: ["中文简体", "English"]      //语言列表，这里不需要qsTr
            delegate: TTextBtn {
                width: langListView.width
                height: 36
                text: modelData
                color: containsMouse ? "lightgray" : pop.barColor
                onClicked: {
                    pop.hide()
                    //调用Q_INVOKABLE导出的函数，reTrans
                    view.reTrans(modelData)
                }
            }
        }
    }
}

```
## 多国语言版本

前面的代码只有中文和英文，已经可以说明问题了。

这里再把最终版本的多国语言的贴出来。

这是头文件
```c++
//TaoView.h
#pragma once

#include <QQuickView>
#include <memory>

class TaoView : public QQuickView
{
    Q_OBJECT
    Q_PROPERTY(QStringList languageList READ languageList NOTIFY languageListChanged)
public:
    explicit TaoView(QWindow *parent = nullptr);
    Q_INVOKABLE void reTrans(const QString &lang);
    const QStringList &languageList() const
    {
        return m_languageList;
    }

signals:
    void reTransed();
    void languageListChanged();

private:
    QString m_lang;
    QMap<QString, std::shared_ptr<QTranslator>> m_transMap;
    QTranslator *m_pLastLang = nullptr;
    QStringList m_languageList;
};
```

这是cpp

```c++
//TaoView.cpp
#include "stdafx.h"

#include "TaoView.h"

#include <QTranslator>
#include <QQmlEngine>
TaoView::TaoView(QWindow *parent) : QQuickView(parent)
{
    setFlag(Qt::FramelessWindowHint);
    setResizeMode(SizeRootObjectToView);
    setColor(QColor(Qt::transparent));

    m_languageList << u8"中文简体"
                   << u8"English"
                   << u8"日本語"
                   << u8"한국어"
                   << u8"Français"
                   << u8"Español"
                   << u8"Portugués"
                   << u8"In Italiano"
                   << u8"русский язык"
                   << u8"Tiếng Việt"
                   << u8"Deutsch"
                   << u8" عربي ، ";
    QStringList fileList;
    fileList << "trans_zh.qm"
            << "trans_en.qm"
            << "trans_ja.qm"
            << "trans_ko.qm"
            << "trans_fr.qm"
            << "trans_es.qm"
            << "trans_pt.qm"
            << "trans_it.qm"
            << "trans_ru.qm"
            << "trans_vi.qm"
            << "trans_de.qm"
            << "trans_ar.qm";

    for (auto i = 0; i < m_languageList.length(); ++i)
    {
        auto trans = std::make_shared<QTranslator>();
        bool ok = trans->load(fileList.at(i));
        qWarning() << m_languageList.at(i) << fileList.at(i) << ok;
        m_transMap[m_languageList.at(i)] = trans;
    }
    m_pLastLang = m_transMap[m_languageList.at(0)].get();
    QCoreApplication::installTranslator(m_pLastLang);
    m_lang = m_languageList.at(0);
    emit languageListChanged();
}

void TaoView::reTrans(const QString &lang)
{
    if (m_lang == lang)
    {
        return;
    }
    m_lang = lang;

    QCoreApplication::removeTranslator(m_pLastLang);
    m_pLastLang = m_transMap[lang].get();
    QCoreApplication::installTranslator(m_pLastLang);
    engine()->retranslate();
    emit reTransed();
}
```

还有qml
```qml
    ....
        TImageBtn {
            width: 20
            height: 20
            anchors.verticalCenter: parent.verticalCenter
            imageUrl: containsMouse ? "qrc:/Image/Window/lang_white.png" : "qrc:/Image/Window/lang_gray.png"
            onClicked: {
                pop.show()
            }
            TPopup {
                id: pop
                barColor: gConfig.reserverColor
                backgroundWidth: 100
                backgroundHeight: 400
                contentItem: ListView {
                    id: langListView
                    anchors.fill: parent
                    anchors.margins: 2
                    model: view.languageList        //这里换成了list属性
                    delegate: TTextBtn {
                        width: langListView.width
                        height: 36
                        text: modelData
                        color: containsMouse ? "lightgray" : pop.barColor
                        onClicked: {
                            pop.hide()
                            view.reTrans(modelData)
                        }
                    }
                }
            }
        }
    ....
```