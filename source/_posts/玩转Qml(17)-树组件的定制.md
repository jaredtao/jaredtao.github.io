---
title: 玩转Qml(17)-树组件的定制
photos: /images/Qml13/1.png
tags:
  - Qt
  - Qml
  - QtQuick
  - 组件化编程
categories: 玩转Qml
abbrlink: 18621
date: 2020-06-15 23:44:23
---
- [简介](#简介)
- [发行说明](#发行说明)
- [效果预览](#效果预览)
- [Qt本身的国际化](#qt本身的国际化)
- [存在翻译不全的问题](#存在翻译不全的问题)
- [新的方案](#新的方案)
- [关于批量翻译](#关于批量翻译)
- [总结](#总结)

## 简介

最近遇到一些需求，要在Qt/Qml中开发树结构，并能够导入、导出json格式。

于是我写了一个简易的Demo，并做了一些性能测试。

在这里将源码、实现原理、以及性能测试都记录、分享出来，算是抛砖引玉吧，希望有更多人来讨论、交流。


## TreeEdit源码

起初的代码在单独的仓库

github https://github.com/jaredtao/TreeEdit

gitee镜像 https://gitee.com/jaredtao/Tree

后续会收录到《玩转Qml》配套的开源项目TaoQuick中

github https://github.com/jaredtao/TaoQuick

gitee镜像 https://gitee.com/jaredtao/TaoQuick


## 效果预览

看一下最终效果

![预览](/images/Qml13/1.png)

Qml实现的树结构编辑器, 功能包括:

树结构的缩进
节点展开、折叠
添加节点
删除节点
重命名节点
搜索
导入
导出
节点属性编辑（完善中）

## 原理说明

数据model的实现，使用C++，继承于QAbstractListModel，并实现rowCount、data等方法。

model本身是List结构的，在此基础上，对model数据进行扩展以模拟树结构，例如增加了 “节点深度”、“是否有子节点”等数据段。

view使用Qml Controls 2中的ListView模拟实现(Controls 1 中的TreeView即将被废弃)。


## 关键代码
### model

基本model的声明如下:
```C++
template <typename T>
class TaoListModel : public QAbstractListModel {
public:
    //声明父类
    using Super = QAbstractListModel;

    TaoListModel(QObject* parent = nullptr);
    TaoListModel(const QList<T>& nodeList, QObject* parent = nullptr);

    const QList<T>& nodeList() const
    {
        return m_nodeList;
    }
    void setNodeList(const QList<T>& nodeList);

    int rowCount(const QModelIndex& parent) const override;

    QVariant data(const QModelIndex& index, int role) const override;
    bool setData(const QModelIndex& index, const QVariant& value, int role) override;


    bool insertRows(int row, int count, const QModelIndex& parent = QModelIndex()) override;
    bool removeRows(int row, int count, const QModelIndex& parent = QModelIndex()) override;

    Qt::ItemFlags flags(const QModelIndex& index) const override;
    Qt::DropActions supportedDropActions() const override;

protected:
    QList<T> m_nodeList;
};
```
其中数据成员使用 QList m_nodeList 存储, 大部分成员函数是对此数据的操作。

Json格式的model声明如下：

```C++
const static QString cDepthKey = QStringLiteral("TModel_depth");
const static QString cExpendKey = QStringLiteral("TModel_expend");
const static QString cChildrenExpendKey = QStringLiteral("TModel_childrenExpend");
const static QString cHasChildendKey = QStringLiteral("TModel_hasChildren");
const static QString cParentKey = QStringLiteral("TModel_parent");
const static QString cChildrenKey = QStringLiteral("TModel_children");

const static QString cRecursionKey = QStringLiteral("subType");
const static QStringList cFilterKeyList = { cDepthKey, cExpendKey, cChildrenExpendKey, cHasChildendKey, cParentKey, cChildrenKey };
class TaoJsonTreeModel : public TaoListModel<QJsonObject> {
    Q_OBJECT
    Q_PROPERTY(int count READ count NOTIFY countChanged)
public:
    //声明父类
    using Super = TaoListModel<QJsonObject>;
    //从json文件读入数据
    Q_INVOKABLE void loadFromJson(const QString& jsonPath, const QString& recursionKey = cRecursionKey);
    //导出到json文件
    Q_INVOKABLE bool saveToJson(const QString& jsonPath, bool compact = false) const;
    Q_INVOKABLE void clear();
    //设置指定节点的数值
    Q_INVOKABLE void setNodeValue(int index, const QString &key, const QVariant &value);
    //在index添加子节点。刷新父级，返回新项index
    Q_INVOKABLE int addNode(int index, const QJsonObject& json);
    Q_INVOKABLE int addNode(const QModelIndex& index, const QJsonObject& json)
    {
        return addNode(index.row(), json);
    }
    //删除。递归删除所有子级,刷新父级
    Q_INVOKABLE void remove(int index);
    Q_INVOKABLE void remove(const QModelIndex& index)
    {
        remove(index.row());
    }
    Q_INVOKABLE QList<int> search(const QString& key, const QString& value, Qt::CaseSensitivity cs = Qt::CaseInsensitive) const;
    //展开子级。只展开一级,不递归
    Q_INVOKABLE void expand(int index);
    Q_INVOKABLE void expand(const QModelIndex& index)
    {
        expand(index.row());
    }
    //折叠子级。递归全部子级。
    Q_INVOKABLE void collapse(int index);
    Q_INVOKABLE void collapse(const QModelIndex& index)
    {
        collapse(index.row());
    }
    //展开到指定项。递归
    Q_INVOKABLE void expandTo(int index);
    Q_INVOKABLE void expandTo(const QModelIndex& index)
    {
        expandTo(index.row());
    }
    //展开全部
    Q_INVOKABLE void expandAll();

    //折叠全部
    Q_INVOKABLE void collapseAll();


    int count() const;

    Q_INVOKABLE QVariant data(int idx, int role = Qt::DisplayRole) const
    {
        return Super::data(Super::index(idx), role);
    }
signals:
    void countChanged();
    ...
};
```

TaoJsonTreeModel继承于TaoListModel，并提供大量Q_INVOKABLE函数，以供Qml调用。

### view

TreeView的模拟实现如下：

```qml
Item {
    id: root
    readonly property string __depthKey: "TModel_depth"
    readonly property string __expendKey: "TModel_expend"
    readonly property string __childrenExpendKey: "TModel_childrenExpend"
    readonly property string __hasChildendKey: "TModel_hasChildren"

    readonly property string __parentKey: "TModel_parent"
    readonly property string __childrenKey: "TModel_children"
    ...
    ListView {
        id: listView
        anchors.fill: parent
        currentIndex: -1
        delegate: Rectangle {
            id: delegateRect
            width: listView.width
            color: (listView.currentIndex === index || area.hovered) ? config.normalColor : config.darkerColor
            // 根据 expaned 判断是否展开，不展开的情况下高度为0
            height: model.display[__expendKey] === true ? 35 : 0
            // 优化。高度为0时visible为false，不渲染。
            visible: height > 0
            property alias editable: nameEdit.editable
            property alias editItem: nameEdit
            TTextInput {
                id: nameEdit
                anchors.verticalCenter: parent.verticalCenter
                //按深度缩进
                x: root.basePadding + model.display[__depthKey] * root.subPadding
                text: model.display["name"]
                height: parent.height
                width: parent.width * 0.8 - x
                editable: false
                onTEditFinished: {
                    sourceModel.setNodeValue(index, "name", displayText)
                }
            }
            TTransArea {
                id: area
                height: parent.height
                width: parent.width - controlIcon.x
                hoverEnabled: true
                acceptedButtons: Qt.LeftButton | Qt.RightButton
                onPressed: {
                    //单击时切换当前选中项
                    if (listView.currentIndex !== index) {
                        listView.currentIndex = index;
                    } else {
                        listView.currentIndex = -1;
                    }
                }
                onTDoubleClicked: {
                    //双击进入编辑状态
                    delegateRect.editable = true;
                    nameEdit.forceActiveFocus()
                    nameEdit.ensureVisible(0)
                }
            }
            Image {
                id: controlIcon
                anchors {
                    verticalCenter: parent.verticalCenter
                    right: parent.right
                    rightMargin: 20
                }
                //有子节点时，显示小图标
                visible: model.display[__hasChildendKey]
                source: model.display[__childrenExpendKey] ? "qrc:/img/collapse.png" : "qrc:/img/expand.png"
                MouseArea {
                    anchors.fill: parent
                    onClicked: {
                        //点击小图标时，切换折叠、展开的状态
                        if (model.display[__hasChildendKey]) {
                            if( true === model.display[__childrenExpendKey]) {
                                collapse(index)
                            } else {
                                expand(index)
                            }
                        }
                    }
                }
            }
        }
    }
    ...
}
```
model层并没有扩展role，而是在data函数的role为display时直接返回json数据，

所以delegate中统一使用model.display[xxx]的方式访问数据。

## 性能测试

### 测试环境

CPU: Intel i5-8400 2.8GHz 六核

内存: 16GB

OS: Windows10 1909

Qt: 5.12.6

编译器: msvc 2017 x64

测试框架: QTest

### 测试方法

#### 数据生成

使用node表示根节点的数量，depth表示每个根节点下面嵌套节点的层数。

例如： node 等于 100， depth 等于10，则数据如下：

![预览](/images/Qml13/2.png)

顶层有100个节点，每个节点下面再嵌套10层，共计节点 100 + 100 * 10 = 1100.

生成json数据的代码如下：
```C++
...
//单元测试类
class LoadTest : public QObject {
    Q_OBJECT

public:
    LoadTest();
    ~LoadTest();

    static void genJson(const QPoint& point);

    ...
//私有槽函数会被QTest调用
private slots:
    //初始化
    void initTestCase();
    //清理
    void cleanupTestCase();
    //测试导入
    void test_load();
    //测试导入前，准备数据
    void test_load_data();
    //测试导出
    void test_save();
    //测试导出前，准备数据
    void test_save_data();
};
...
//节点最大值
const int nodeMax = 10000;
//嵌套深度最大值
const int depthMax = 100;

void LoadTest::genJson(const QPoint& point)
{
    using namespace TaoCommon;
    int node = point.x();
    int depth = point.y();
    QJsonArray arr;
    for (int i = 0; i < node; ++i) {
        QJsonObject obj;
        obj["name"] = QString("node_%1").arg(i);
        QVector<QJsonArray> childrenArr = { depth, QJsonArray { QJsonObject {} } };
        //最后一个节点，嵌套层级最深的。
        childrenArr[depth - 1][0] = QJsonObject { { "name", QString("node_%1_%2").arg(i).arg(depth - 1) } };
        //从后往前倒推。
        for (int j = depth - 2; j >= 0; --j) {
            childrenArr[j][0] = QJsonObject { { cRecursionKey, childrenArr[j + 1] }, { "name", QString("node_%1_%2").arg(i).arg(j) } };
        }
        obj[cRecursionKey] = childrenArr[0];
        arr.append(obj);
    }
    writeJsonFile(qApp->applicationDirPath() + QString("/%1_%2.json").arg(node).arg(depth), arr);
}

void LoadTest::initTestCase()
{
    QList<QPoint> list;
    for (int i = 1; i <= nodeMax; i *= 10) {
        for (int j = 1; j <= depthMax; j *= 10) {
            list.append({ i, j });
        }
    }
    auto result = QtConcurrent::map(list, &LoadTest::genJson);
    result.waitForFinished();
}
```
初始化函数initTestCase中，组织了一个QList，然后使用QtConcurrent::map并发调用genJson函数，生成数据json文件。

node和depth每次扩大10倍。

经过测试，嵌套层数在100以上时，Qt可能会崩溃。要么是QJsonDocument无法解析，要么是Qml挂掉。所以不使用100以上的嵌套级别。

### 测试过程

QTest十分好用，简单易上手，参考帮助文件即可

例如测试加载的代码如下:

```C++
void LoadTest::prepareData()
{
    //添加两列数据
    QTest::addColumn<int>("node");
    QTest::addColumn<int>("depth");
    //添加行
    for (int i = 1; i <= nodeMax; i *= 10) {
        for (int j = 1; j <= depthMax; j *= 10) {
            QTest::newRow(QString("%1_%2").arg(i).arg(j).toStdString().c_str()) << i << j;
        }
    }
}
void LoadTest::test_load_data()
{
    //准备数据
    prepareData();
}
void LoadTest::test_load()
{
    using namespace TaoCommon;
    //取数据
    QFETCH(int, node);
    QFETCH(int, depth);
    TaoJsonTreeModel model;
    //性能测试
    QBENCHMARK
    {
        model.loadFromJson(qApp->applicationDirPath() + QString("/%1_%2.json").arg(node).arg(depth));
    }
}
```

### 测试结果

![预览](/images/Qml13/3.png)

一秒内最多可以加载的数据量在十万级别，包括

10000 x 10耗时在 386毫秒，1000 x 100 耗时在671毫秒。