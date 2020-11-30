---
title: QQuickWidget中文输入法问题的正确解法
photos: /img/avatar.jpg
tags:
  - Qt
  - Qt实用技能
categories: 玩转Qt
abbrlink: 62819
date: 2020-11-30 12:44:23
---

- [QQuickWidget中文输入法问题的正确解法](#qquickwidget中文输入法问题的正确解法)
  - [Qt的bug](#qt的bug)
  - [旧的解法](#旧的解法)
  - [正确的解法](#正确的解法)

# QQuickWidget中文输入法问题的正确解法

本文分享特定问题的解法,用不到的可以忽略。

## Qt的bug

使用QQuickWidget的时候，遇到过这个问题：界面的TextInput 或者TextEdit, 鼠标点击聚焦后，切换为光标输入状态，此时切换系统中文输入法，会发现无法输入。

(系统任务栏的输入法状态是正确的,界面上输入字符，直接显示英文，无法显示输入法的候选框)

需要把界面切到其它软件，再切换回来，之后就能够输入了。

可以参考Qt官方bug报告:

https://bugreports.qt.io/browse/QTBUG-61475

## 旧的解法

这个Bug是2018年报告的，我们当时做项目，也被这个Bug坑到了。

当时我给出了一个弱化版本的解法，原理是在第一次聚焦的时候，清理掉QQuickWidget的焦点。

```C++
QuickWidget::QuickWidget(QWidget *parent)
    : QQuickWidget(parent)
{
    ...
    connect(quickWindow(), &QQuickWindow::activeFocusItemChanged, this, &QuickWidget::onClearFocus);
    ...
}
void QuickWidget::onClearFocus()
{
    QQuickItem *pItem = quickWindow()->activeFocusItem();
    if (pItem && (pItem->inherits("QQuickTextInput") || pItem->inherits("QQuickTextField"))) 
    {
        disconnect(quickWindow(), &QQuickWindow::activeFocusItemChanged, this, &QuickWidget::onClearFocus);
        QuickWidget::clearFocus();
    }
}
```

此方法勉强能用，一些细节上体验不太好。

当时找不到更好的方法，就这样用着了。

## 正确的解法

2020年Qt官方终于派出了资深的专家，在Qt5.15.2中，彻底解决了这个问题。

(看到有不少博客、论坛，还在流传我提供的旧版本，于心不忍)

于是我从新版本里面，提炼出来了代码，给使用旧版本的同学解决此问题。

```C++
QuickWidget::QuickWidget(QWidget *parent)
    : QQuickWidget(parent)
{
  ...

#if QT_VERSION < QT_VERSION_CHECK(5, 15, 2)
    connect(quickWindow(), &QQuickWindow::focusObjectChanged, this, &QuickWidget::propagateFocusObjectChanged);
#endif
  
  ...
}

#if QT_VERSION < QT_VERSION_CHECK(5, 15, 2)
bool QuickWidget::event(QEvent *e)
{
    switch (e->type())
    {
        case QEvent::FocusAboutToChange:
            return QCoreApplication::sendEvent(quickWindow(), e);
        default:
            break;
    }
    return Super::event(e);
}
void QuickWidget::propagateFocusObjectChanged(QObject *focusObject)
{
    if (QApplication::focusObject() != this)
        return;
    if (this->window()->windowHandle()) {
        emit this->window()->windowHandle()->focusObjectChanged(focusObject);
    }
}
#endif
```