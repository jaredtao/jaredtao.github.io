---
title: Qt实用技能1-用好QtCreator
photos: /img/avatar.jpg
tags:
  - Qt
  - QtCreator
  - Qt实用技能
categories: Qt进阶之路
abbrlink: 21946
date: 2019-05-18 18:44:23
---

- [简介](#简介)
- [环境说明](#环境说明)
- [QtCreator折叠全部代码](#qtcreator折叠全部代码)
- [QtCreator属性生成](#qtcreator属性生成)
- [QtCreator注释代码](#qtcreator注释代码)
- [QtCreator代码片段](#qtcreator代码片段)
- [QtCreator代码格式化](#qtcreator代码格式化)
- [QtCreator会话管理](#qtcreator会话管理)
- [结尾](#结尾)

## 简介

本文是《Qt实用技能》系列文章的第一篇，涛哥将教大家，一些QtCreator的实用技巧。

工欲善其事，必先利其器。

这个系列，全是干货！

注：文章主要发布在[涛哥的博客](https://jaredtao.github.io) 和 [涛哥的知乎专栏-Qt进阶之路](https://zhuanlan.zhihu.com/TaoQt)

## 环境说明

下文以Windows平台的QtCreator为参考，其它平台的菜单栏入口和快捷键 请以实际为准。

QtCreator版本以Qt5.6.x及以上安装包所带的都行，再旧的版本不讨论。

## QtCreator折叠全部代码

折叠全部代码,支持C++和Qml。操作方式为：

光标焦点放在代码文本中，之后 菜单栏: 编辑->Advanced->Toggle Fold All

这个功能没有快捷键

## QtCreator属性生成

经常需要给自定义的QObject类写一些属性，QtCreator是可以自动生成get、set函数以及change信号的。

只要写上Q_PROPERTY那一行，光标放在Q_PROPERTY上, 用右键菜单 -> Refactor -> Generate Missing Q_PROPERTY Memory 即可生成。

也可以使用快捷键，光标放在Q_PROPERTY上，按Alt + Enter。

![预览](/images/Qml3/1.gif)

## QtCreator注释代码

快捷键，注释当前行代码或者当前选中的多行代码

Ctrl + /

已经注释掉的，再按一次取消注释。


## QtCreator代码片段

前面的Q_PROPERTY自动生成，其实就是一种代码片段。

比如经常要写这样一段代码
```C++
if (pObj) {
    delete pObj;
    pObj = nullptr;
}
```
其中的pObj出现了多次，在不同的地方只是pObj这个名字不同，其它if 和 delete操作一模一样。

可以把这段代码封装成模板函数，也可以做成QtCreator的代码片段。(不建议定义宏, 不类型安全和不方便调试)

模板是这样的：
```C++
template<class T>
safeDelete (T *pObj) 
{
    if (pObj) 
    {
        delete pObj;
        pObj = nullptr;
    }
}
```
代码片段是在写代码时就把实际的代码生成出来了，模板是编译的时候才去生成。所以代码片段可以加快编译速度。

代码片段是这么做的，在菜单的 工具->选项 弹出选项窗口，然后到文本编辑器->片段->下拉选C++

![预览](/images/Qt1/3.png)

添加一个片段，起名字叫safeD，并填上内容
```c++
if ($$) {
    delete $$;
    $$ = nullptr;
}
```
![预览](/images/Qt1/4.png)

写好后点击确定。再回到代码中，输入safeD，按回车就会自动补全前面的片段。

光标出现在$$的地方，有一个高亮颜色，此时只要输入一个名字，按下回车键，后续的地方自动替换成输入的名字了。

![预览](/images/Qt1/5.gif)

## QtCreator代码格式化

都9102年了，如果还有人跟你计较大括号要不要换行、指针符号靠左还是靠右这种问题，请用自动格式化工具怼他/她。

QtCreator支持很多种格式化工具，涛哥用的是clang-format。VisualStudio 2017、2019、以及VSCode也支持clang-format

的,都不需要额外安装任何插件。配置起来很简单，只要在项目pro文件同级目录下，放一个配置好的.clang-format的文件就行了。

团队合作的时候，使用同一个.clang-format配置文件，大家的代码格式就都一致了。

clang-format有默认的google、llvm等格式可选，也可以自定义。下面是一个涛哥使用的自定义配置文件，并做了详细的注释

```clang-format
---
# 语言: None, Cpp, Java, JavaScript, ObjC, Proto, TableGen, TextProto
Language:        Cpp
# BasedOnStyle:  WebKit
# 访问说明符(public、private等)的偏移
AccessModifierOffset: -4
# 开括号(开圆括号、开尖括号、开方括号)后的对齐: Align, DontAlign, AlwaysBreak(总是在开括号后换行)
AlignAfterOpenBracket: AlwaysBreak
# 连续赋值时，对齐所有等号
AlignConsecutiveAssignments: false
# 连续声明时，对齐所有声明的变量名
AlignConsecutiveDeclarations: false
# 左对齐逃脱换行(使用反斜杠换行)的反斜杠
AlignEscapedNewlines: Right
# 水平对齐二元和三元表达式的操作数
AlignOperands:   true
# 对齐连续的尾随的注释
AlignTrailingComments: true
# 允许函数声明的所有参数在放在下一行
AllowAllParametersOfDeclarationOnNextLine: true
# 允许短的块放在同一行
AllowShortBlocksOnASingleLine: false
# 允许短的case标签放在同一行
AllowShortCaseLabelsOnASingleLine: false
# 允许短的函数放在同一行: None, InlineOnly(定义在类中), Empty(空函数), Inline(定义在类中，空函数), All
AllowShortFunctionsOnASingleLine: Empty
# 允许短的if语句保持在同一行
AllowShortIfStatementsOnASingleLine: false
# 允许短的循环保持在同一行
AllowShortLoopsOnASingleLine: false
# 总是在定义返回类型后换行(deprecated)
AlwaysBreakAfterDefinitionReturnType: None
# 总是在返回类型后换行: None, All, TopLevel(顶级函数，不包括在类中的函数),
#   AllDefinitions(所有的定义，不包括声明), TopLevelDefinitions(所有的顶级函数的定义)
AlwaysBreakAfterReturnType: None
# 总是在多行string字面量前换行
AlwaysBreakBeforeMultilineStrings: false
# 总是在template声明后换行
AlwaysBreakTemplateDeclarations: true
# false表示函数实参要么都在同一行，要么都各自一行
BinPackArguments: false
# false表示所有形参要么都在同一行，要么都各自一行
BinPackParameters: false
# 大括号换行，只有当BreakBeforeBraces设置为Custom时才有效
BraceWrapping:   
  # class定义后面
  AfterClass:      true
  # 控制语句后面
  AfterControlStatement: true
  # enum定义后面
  AfterEnum:       true
  # 函数定义后面
  AfterFunction:   true
  # 命名空间定义后面
  AfterNamespace:  true
  # ObjC定义后面
  AfterObjCDeclaration: false
  # struct定义后面
  AfterStruct:     true
  # union定义后面
  AfterUnion:      true
  # extern 定义后面
  AfterExternBlock: true
  # catch之前
  BeforeCatch:     true
  # else 之前
  BeforeElse:      true
  # 缩进大括号
  IndentBraces:    false

  SplitEmptyFunction: true

  SplitEmptyRecord: true

  SplitEmptyNamespace: true
# 在二元运算符前换行: None(在操作符后换行), NonAssignment(在非赋值的操作符前换行), All(在操作符前换行)
BreakBeforeBinaryOperators: All
# 在大括号前换行: Attach(始终将大括号附加到周围的上下文), Linux(除函数、命名空间和类定义，与Attach类似),
#   Mozilla(除枚举、函数、记录定义，与Attach类似), Stroustrup(除函数定义、catch、else，与Attach类似),
#   Allman(总是在大括号前换行), GNU(总是在大括号前换行，并对于控制语句的大括号增加额外的缩进), WebKit(在函数前换行), Custom
#   注：这里认为语句块也属于函数
BreakBeforeBraces: Allman
# 继承列表的逗号前换行
BreakBeforeInheritanceComma: false
# 在三元运算符前换行
BreakBeforeTernaryOperators: true
# 在构造函数的初始化列表的逗号前换行
BreakConstructorInitializersBeforeComma: false
# 初始化列表前换行
BreakConstructorInitializers: BeforeComma
# Java注解后换行
BreakAfterJavaFieldAnnotations: false

BreakStringLiterals: true
# 每行字符的限制，0表示没有限制
ColumnLimit:     160
# 描述具有特殊意义的注释的正则表达式，它不应该被分割为多行或以其它方式改变
CommentPragmas:  '^ IWYU pragma:'
# 紧凑 命名空间
CompactNamespaces: false
# 构造函数的初始化列表要么都在同一行，要么都各自一行
ConstructorInitializerAllOnOneLineOrOnePerLine: true
# 构造函数的初始化列表的缩进宽度
ConstructorInitializerIndentWidth: 4
# 延续的行的缩进宽度
ContinuationIndentWidth: 4
# 去除C++11的列表初始化的大括号{后和}前的空格
Cpp11BracedListStyle: false
# 继承最常用的指针和引用的对齐方式
DerivePointerAlignment: false
# 关闭格式化
DisableFormat:   false
# 自动检测函数的调用和定义是否被格式为每行一个参数(Experimental)
ExperimentalAutoDetectBinPacking: false
# 固定命名空间注释
FixNamespaceComments: true
# 需要被解读为foreach循环而不是函数调用的宏
ForEachMacros:   
  - foreach
  - Q_FOREACH
  - BOOST_FOREACH

IncludeBlocks:   Preserve
# 对#include进行排序，匹配了某正则表达式的#include拥有对应的优先级，匹配不到的则默认优先级为INT_MAX(优先级越小排序越靠前)，
#   可以定义负数优先级从而保证某些#include永远在最前面
IncludeCategories: 
  - Regex:           '^"(llvm|llvm-c|clang|clang-c)/'
    Priority:        2
  - Regex:           '^(<|"(gtest|gmock|isl|json)/)'
    Priority:        3
  - Regex:           '.*'
    Priority:        1
IncludeIsMainRegex: '(Test)?$'
# 缩进case标签
IndentCaseLabels: true

IndentPPDirectives: None
# 缩进宽度
IndentWidth:     4
# 函数返回类型换行时，缩进函数声明或函数定义的函数名
IndentWrappedFunctionNames: false

JavaScriptQuotes: Leave

JavaScriptWrapImports: true
# 保留在块开始处的空行
KeepEmptyLinesAtTheStartOfBlocks: true
# 开始一个块的宏的正则表达式
MacroBlockBegin: ''
# 结束一个块的宏的正则表达式
MacroBlockEnd:   ''
# 连续空行的最大数量
MaxEmptyLinesToKeep: 1

# 命名空间的缩进: None, Inner(缩进嵌套的命名空间中的内容), All
NamespaceIndentation: All
# 使用ObjC块时缩进宽度
ObjCBlockIndentWidth: 4
# 在ObjC的@property后添加一个空格
ObjCSpaceAfterProperty: true
# 在ObjC的protocol列表前添加一个空格
ObjCSpaceBeforeProtocolList: true

PenaltyBreakAssignment: 2

PenaltyBreakBeforeFirstCallParameter: 19
# 在一个注释中引入换行的penalty
PenaltyBreakComment: 300
# 第一次在<<前换行的penalty
PenaltyBreakFirstLessLess: 120
# 在一个字符串字面量中引入换行的penalty
PenaltyBreakString: 1000
# 对于每个在行字符数限制之外的字符的penalty
PenaltyExcessCharacter: 1000000
# 将函数的返回类型放到它自己的行的penalty
PenaltyReturnTypeOnItsOwnLine: 60
# 指针和引用的对齐: Left, Right, Middle
PointerAlignment: Right

#RawStringFormats: 
#  - Delimiter:       pb
#    Language:        TextProto
#    BasedOnStyle:    google
# 允许重新排版注释
ReflowComments:  false
# 允许排序#include
SortIncludes:    true

SortUsingDeclarations: true
# 在C风格类型转换后添加空格
SpaceAfterCStyleCast: false
# 模板关键字后面添加空格
SpaceAfterTemplateKeyword: true
# 在赋值运算符之前添加空格
SpaceBeforeAssignmentOperators: true
# 开圆括号之前添加一个空格: Never, ControlStatements, Always
SpaceBeforeParens: ControlStatements
# 在空的圆括号中添加空格
SpaceInEmptyParentheses: false
# 在尾随的评论前添加的空格数(只适用于//)
SpacesBeforeTrailingComments: 1
# 在尖括号的<后和>前添加空格
SpacesInAngles:  false
# 在容器(ObjC和JavaScript的数组和字典等)字面量中添加空格
SpacesInContainerLiterals: true
# 在C风格类型转换的括号中添加空格
SpacesInCStyleCastParentheses: false
# 在圆括号的(后和)前添加空格
SpacesInParentheses: false
# 在方括号的[后和]前添加空格，lamda表达式和未指明大小的数组的声明不受影响
SpacesInSquareBrackets: false
# 标准: Cpp03, Cpp11, Auto
Standard:        Cpp11
# tab宽度
TabWidth:        4
# 使用tab字符: Never, ForIndentation, ForContinuationAndIndentation, Always
UseTab:          Never
...
```
怎么使用呢？QtCreator中打开要格式化的代码文件，按快捷键 Ctrl + I 就是格式化当前行。

Ctrl + A选中全部内容，再按Ctrl + I就是格式化全部。

VS是把.clang-format文件放在和sln文件同级目录即可，快捷键一般是 先按Ctrl + K 再按Ctrl + D

VSCode是打开的文件夹根目录有.clang-format即可，格式化一般在右键菜单。

## QtCreator会话管理

QtCreator的"会话管理"和"最近使用"功能配合，是涛哥接触过的所有IDE/Editor中，管理项目最好用的，没有之一。

包括VisualStudio、VSCode、AndroidStudio、XCode、Unity3D Editor等等，都只有"最近使用"，没有"会话管理"。

![预览](/images/Qt1/6.png)

如上图，左边是会话列表，右边是最近使用列表。

会话管理的最大用处是，同时打开多个Qt项目，以及快速切换并还原状态。

涛哥用示例来说明:

涛哥正在开发TaoQuick项目，这个项目包含两个不同路径下的pro项目, 每个项目分别有自己的子项目。

![预览](/images/Qt1/session1.png)

当我正在调试TaoView.cpp文件，并且打了断点的时候，有小伙伴来问我关于另一个项目HelloCI的一些问题。

这时候我需要把代码切换到HelloCI项目，有些人可能会想着再打开一个QtCreator，当然这样也行，就是

窗口太多了容易搞混了。涛哥更信赖“会话管理”功能，切换到HelloCI这个会话，做了一些处理。

完了之后，涛哥又切换回了TaoQuick这个会话，QtCreator就自动恢复到了刚才看的代码TaoView.cpp，而且断点也还在。

又过了一段时间，涛哥需要重启一下电脑，重启后打开QtCreator，直接点开TaoQuick这个会话，又给我切换回

刚才调试的TaoQuick.cpp文件了，项目结构展开和重启之前是一样的,只有断点没有了。

简而言之，大家把平常用的多个相关的项目，放进一个会话里面，就可以放心地关掉QtCreator。下一次想打开的时候，只要点一下会话就可以了。

![预览](/images/Qt1/7.gif)

## 结尾

这次就分享这么多了，以上内容，大部分都可以在TaoQuick的代码仓库中看到

[https://github.com/jaredtao/taoquick](https://github.com/jaredtao/taoquick)