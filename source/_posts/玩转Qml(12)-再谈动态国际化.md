---
title: 玩转Qml(12)-再谈动态国际化
photos: /images/Qml12/1.gif
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: 玩转Qml
abbrlink: 18621
date: 2019-06-03 01:44:23
---
- [简介](#简介)
- [源码](#源码)
- [效果预览](#效果预览)
- [Qt本身的国际化](#qt本身的国际化)
- [存在翻译不全的问题](#存在翻译不全的问题)
- [新的方案](#新的方案)
- [关于批量翻译](#关于批量翻译)
- [总结](#总结)

## 简介

本文是《玩转Qml》系列文章的第十二篇，主要讨论多国语言动态翻译。

之前分享过使用Qt自带翻译的方案，但是效果不太好。这次分享一个非官方的多国语言方案。

## 源码

《玩转Qml》系列文章，配套了一个优秀的开源项目:TaoQuick

github https://github.com/jaredtao/TaoQuick

访问不了或者速度太慢，可以用国内的镜像网站gitee

https://gitee.com/jaredtao/TaoQuick

## 效果预览

看一下最终效果

![预览](/images/Qml12/1.gif)

(原始字符串全部为英文，中文为人工翻译。

其它语言使用的百度翻译api批量翻译，不太准确，暂时先这样)

## Qt本身的国际化

先来回顾一下，Qt的国际化方案：

1. C++代码中的字符串使用QObject::tr()包起来，类本身是QObject的子类时可以省略作用域“QObject::”,直接写tr
   
2. qml代码中使用qsTr把字符串包起来

3. pro文件中添加一句TRANSLATIONS += trans_zh.qs ，这个名字起什么无所谓，关键是‘_zh’要有。
   
4. 调用lrelease工具,扫描项目并生成trans_zh.qs 文件。这个文件是xml格式的，未经过翻译的，需要为这个文件做一些翻译工作。

5. 翻译做好后，调用lupdate工具，生成trans_zh.qm文件。这个文件就是把xml压缩成了二进制。
    
6. 将qm文件放在运行路径，或者资源文件里。
   
7. 切换语言时， Qt/C++代码中使用QTranslater加载qm文件，QCoreApplication卸载旧的QTranslater，并安装新的QTranslater。调用
   QmlEngine::retranslate函数

在5.10以前的版本，Qt是不能直接动态切换语言的，要么重新启动程序，要么把所有的text都set一遍，retranslate是5.10才有的接口。

## 存在翻译不全的问题

上面的方案，在[TaoQuick](https://github.com/jaredtao/TaoQuick)中使用了。

明显的问题是，只能翻译静态的内容，动态加载的ListModel，动态切换语言时不能自动刷新。

按照Qt文档所说，Array或者其它数据结构中的内容，也不能自动刷新。


## 新的方案

这里抛弃Qt的翻译机制，使用自己实现的方案。

1、约定要用到的字符串，全部用英文。

2、翻译文件使用json文件，一个文件翻译一种语言。

文件命名格式language_xx.json, json内容格式如下；

```json
{
    "lang": "简体中文",
    "trans":[
        {
            "key": "Chinese",
            "value": "简体中文"
        },
        {
            "key": "Japanese",
            "value": "日语"
        },
        {
            "key": "Korean",
            "value": "韩语"
        },
        {
            "key": "Menu",
            "value": "菜单"
        },
    ]
}
```

其中lang字段表示当前语言，trans字段是所有的翻译项。

3、实现核心翻译器Trans

自己实现一个Trans类，用来加载翻译包、提供翻译数据，类声明如下:

```c++
//Trans.h
#pragma once

#include <QObject>
#include <QHash>
#include <QList>
#include <QString>
class Trans : public QObject
{
    Q_OBJECT
    //当前语言
    Q_PROPERTY(QString currentLang READ currentLang WRITE setCurrentLang NOTIFY currentLangChanged)
    //支持的语言列表
    Q_PROPERTY(QStringList languages READ languages NOTIFY languagesChanged)
    //空字符串。用于动态翻译时，通过change信号触发trans
    Q_PROPERTY(QString transString READ transString NOTIFY transStringChanged)
public:
    explicit Trans(QObject *parent = nullptr);
    
    //加载指定文件夹
    void loadFolder(const QString &folder);

    //加载指定文件。成功时返回true，lang参数输出文件代表的语言。
    bool load(QString &lang, const QString &filePath);
public:
    const QString &currentLang() const;

    const QStringList &languages() const;

    const QString &transString() const;

public slots:
    //翻译
    QString trans(const QString &source) const;

    void setCurrentLang(const QString &currentLang);
signals:
    void currentLangChanged(const QString &currentLang);

    void languagesChanged(const QStringList &languages);

    void transStringChanged();
protected:
    void setLanguages(const QStringList &languages);

    void initEnglish();
private:
    QString m_currentLang;
    // <"English", <"key", "value">>
    QHash<QString, QHash<QString, QString>> m_map;
    QStringList m_languages;
    //always empty
    QString m_transString;
};

```

其中languages是加载过后支持的所有语言，currentLang是当前语言。

trans函数是用来做翻译的，传入要翻译的字符串，根据当前语言，返回翻译后的字符串。

因为软件到处都要翻译，所以trans函数会被频繁调用，使用QHash<QString, QHash<QString, QString>>这样的

嵌套Hash数据结构，保证查询的平均复杂度为O(1).

transString是一个特殊的属性，其值始终为空，在语言被切换时，会触发transStringChange信号。

这样有什么用呢？先知道这个设定，后面qml部分会详细解释。

cpp 实现如下：

```c++
//Trans.cpp
#include "Trans.h"
#include "FileReadWrite.h"
#include <QDir>
const static auto cEnglisthStr = QStringLiteral("English");
const static auto cChineseStr = QStringLiteral("简体中文");
Trans::Trans(QObject* parent)
    : QObject(parent)
{
}

void Trans::loadFolder(const QString& folder)
{
    QDir dir(folder);
    auto infos = dir.entryInfoList({ "language_*.json" }, QDir::Files);
    for (auto info : infos) {
        QString lang;
        load(lang, info.absoluteFilePath());
    }
    initEnglish();
    auto langs = m_map.uniqueKeys();
    if (langs.contains(cChineseStr)) {
        langs.removeAll(cChineseStr);
        langs.push_front(cChineseStr);
    }
    setLanguages(langs);
    if (m_map.contains(cChineseStr)) {
        setCurrentLang(cChineseStr);
    } else {
        setCurrentLang(cEnglisthStr);
    }
    emit transStringChanged();
}

bool Trans::load(QString& lang, const QString& filePath)
{
    lang.clear();
    QJsonObject rootObj;
    if (!TaoCommon::readJsonFile(filePath, rootObj)) {
        return false;
    }
    lang = rootObj.value("lang").toString();
    const auto& trans = rootObj.value("trans").toArray();
    for (auto i : trans) {
        auto transObj = i.toObject();
        QString key = transObj.value("key").toString();
        QString value = transObj.value("value").toString();
        m_map[lang][key] = value;
    }
    return true;
}

const QString &Trans::currentLang() const
{
    return m_currentLang;
}

const QStringList &Trans::languages() const
{
    return m_languages;
}

const QString &Trans::transString() const
{
    return m_transString;
}

void Trans::initEnglish()
{
    if (!m_map.contains(cEnglisthStr)) {
        QHash<QString, QString> map;
        if (m_map.contains(cChineseStr)) {
            map = m_map.value(cChineseStr);
        } else {
            map = m_map.value(m_map.keys().first());
        }
        for (auto key : map.uniqueKeys()) {
            m_map[cEnglisthStr][key] = key;
        }
    }
}

QString Trans::trans(const QString& source) const
{
    return m_map.value(m_currentLang).value(source, source);
}

void Trans::setCurrentLang(const QString& currentLang)
{
    if (m_currentLang == currentLang)
        return;

    m_currentLang = currentLang;
    emit currentLangChanged(m_currentLang);
    emit transStringChanged();
}

void Trans::setLanguages(const QStringList& languages)
{
    if (m_languages == languages)
        return;

    m_languages = languages;
    emit languagesChanged(m_languages);
    emit transStringChanged();
}

```

4、Qml中使用新的翻译语法

qml中的语法如下：

```qml
Text {
    text: trans.trans("Welcome")  + trans.transString
}
```
这是一个很常规的'Qml属性绑定',或者叫'绑定表达式', 这样写了以后，text的值依赖于trans.trans()函数返回值和 transString。

当text依赖的属性发出change信号时，qml引擎会重新对这个表达式求值，并把结果赋值给text。

一般情况下，text的值就是trans的返回值，后面的空值不会影响到结果。

当前语言被改变时，函数没有change信号，而transString属性的change信号会被触发，导致qml引擎会重新对这个表达式求值，

此时会重新调用trans函数，按照新的语言返回翻译结果。

Text组件的text属性变化时，会自己刷新UI。

于是，就实现了动态翻译多国语言。


对于ListModel,就把静态字符串换成动态的变量即可：

```qml
ListView {
    ...
    delegate: Text {
        text: trans.trans(modelData)  + trans.transString
    }
}
```
复杂一些的格式化字符串，也是没有问题的：
```qml
Text {
    text: trans.trans("Today is %1, i feel %2").arg(trans.trans("Sunday")).arg(trans.trans("happy"))  + trans.transString
}
```
对应的翻译文件：

```json
{
    "lang": "简体中文",
    "trans": [
        {
            "key": "Today is %1, i feel %2",
            "value": "今天是%1, 我感觉%2"
        },
        {
            "key": "Sunday",
            "value": "星期天"
        },
        {
            "key": "happy",
            "value": "开心"
        },
        ...
    ]
}
```
## 关于批量翻译

翻译效果不太理想，不过还是可以分享一下方法。

首先是提取出了所有要翻译的字符串：

```json
//key.json
[
    "Chinese",
    "traditional Chinese",
    "Cantonese",
    "classical Chinese",
    "Japanese",
    "Korean",
    "French",
    "Spanish",
    "Thai",
    "Arabic",
    "Russian",
    "Portuguese",
    "German",
    "Italian",
    "Greek",
    "Dutch",
    "Polish",
    "Bulgarian",
    "Estonian",
    "Danish",
    ...
]
```

其次是写了一个PowerShell脚本，逐个调用百度翻译API，并把结果按照前面的json输出。

```powershell
# trans.ps1

# 需要注册百度翻译的app，获得id和secret

$baiduId = "xxxxxxxxxx"
$baiduSecret = "xxxxxxxxx"
# 列出百度翻译 支持的语言
$baiduLangs = @{
    zh="中文";
    # cht="繁体中文";
    yue="粤语";
    wyw="文言文";
    jp="日语";
    kor="韩语";
    fra="法语";
    spa="西班牙语";
    th="泰语";
    ara="阿拉伯语";
    ru="俄语";
    pt="葡萄牙语";
    de="德语";
    it="意大利语";
    el="希腊语";
    nl="荷兰语";
    # pl="波兰语";
    bul="保加利亚语";
    est="爱沙尼亚语";
    dan="丹麦语";
    fin="芬兰语";
    cs="捷克语";
    rom="罗马尼亚语";
    # slo="斯洛文尼亚语";
    # swe="瑞典语";
    hu="匈牙利语";
    # vie="越南语";
}
# 计算MD5
function getHash([string]$source) {
    $stringAsStream = [System.IO.MemoryStream]::new()
    $writer = [System.IO.StreamWriter]::new($stringAsStream)
    $writer.write($source)
    $writer.Flush()
    $stringAsStream.Position = 0
    $hash = Get-FileHash -InputStream $stringAsStream -algorithm MD5
    
    return $hash.Hash.toString().toLower()
}

# 百度翻译的调用
function baiduTrans {
    param(
        [string]$q,
        [string]$from = 'en',
        [string]$to = 'zh'
    
    )
    # 随机盐
    $salt = Get-Random
    # token拼接
    $signtoken = "{0}{1}{2}{3}" -f $baiduId, $q, $salt, $baiduSecret
    # 计算md5
    $signtoken = getHash $signtoken
    # POST body
    $body = @{
        q     = $q
        from  = $from
        to    = $to
        appid = $baiduId
        salt  = $salt
        sign  = $signtoken
    }
    $response = Invoke-RestMethod http://api.fanyi.baidu.com/api/trans/vip/translate -Method Post -Body $body
    
    #return $response.dst.toString()
    if ($null -ne $response.trans_result) {
        return  $response.trans_result[0].dst 
    }
    else {
        return $q
    }
}
# 主函数
function main() {
    # 读取keys.json
    $json = Get-Content 'keys.json' -Encoding  utf8 | ConvertFrom-Json
    # 每种语言都翻译一遍
    foreach ($lang in $baiduLangs.Keys) {
        Write-Host $lang
        $tlang = $baiduLangs[$lang]
        # 目标语言的名称，默认都用中文表示的。这里将名称翻译成对应的语言。比如：英语 就叫'English'， 日语就用'日本語'
        if ($lang -ne "zh") {
            $tlang = baiduTrans $baiduLangs[$lang] "zh" $lang
        }
        # 逐条翻译
        $res=@()
        foreach ($key in $json) {
            Write-Host $key
            $dst = baiduTrans $key "en" $lang
            $t = @{'key'=$key; 'value'=$dst}
            $res +=$t
            # 延时1秒，毕竟没有充值，被限速了，每秒只能请求1次。
            Start-Sleep -Seconds 1
        }
        # 按格式输出
        $obj = @{
            "lang" = $tlang
            "trans" = $res
        }
        $targetFileName = "language_{0}.json" -f $lang
        $obj | ConvertTo-Json | Set-Content $targetFileName -Encoding UTF8
    }
}

# baiduTrans  "apple" "en" "zh"

main
```
结果如下，生成了一堆json文件
![](/images/Qml12/2.png)

## 总结

Qml中带个尾巴的写法，虽然有些别扭，但是够用、能达到动态翻译的目标。

如果你有更好的思路，欢迎留言交流。



