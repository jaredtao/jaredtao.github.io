---
title: 玩转Qt(12)-github-Actions缓存优化
photos: /images/QtActions2/3.png
tags:
  - Qt
  - 持续集成(CI)
categories: 玩转Qt
abbrlink: 26275
date: 2019-12-4 12:44:23
---
- [简介](#%e7%ae%80%e4%bb%8b)
- [原理](#%e5%8e%9f%e7%90%86)
  - [缓存actions模板](#%e7%bc%93%e5%ad%98actions%e6%a8%a1%e6%9d%bf)
  - [缓存文档](#%e7%bc%93%e5%ad%98%e6%96%87%e6%a1%a3)
  - [缓存大小限制](#%e7%bc%93%e5%ad%98%e5%a4%a7%e5%b0%8f%e9%99%90%e5%88%b6)
  - [缓存运作流程](#%e7%bc%93%e5%ad%98%e8%bf%90%e4%bd%9c%e6%b5%81%e7%a8%8b)
- [Qt项目的缓存优化](#qt%e9%a1%b9%e7%9b%ae%e7%9a%84%e7%bc%93%e5%ad%98%e4%bc%98%e5%8c%96)
  - [无缓存的配置](#%e6%97%a0%e7%bc%93%e5%ad%98%e7%9a%84%e9%85%8d%e7%bd%ae)
  - [加缓存](#%e5%8a%a0%e7%bc%93%e5%ad%98)
  - [环境变量还原](#%e7%8e%af%e5%a2%83%e5%8f%98%e9%87%8f%e8%bf%98%e5%8e%9f)
  - [最终配置](#%e6%9c%80%e7%bb%88%e9%85%8d%e7%bd%ae)

## 简介

在之前两篇文章《github-Actions自动化编译》《github-Actions自动化发行》中，

介绍了github-Actions的一些用法，其中有部分配置，已经有了缓存相关的步骤。

这里专门开一篇文章，来记录github-Actions的缓存优化相关的知识。

## 原理

### 缓存actions模板

github-Actions提供了缓存模板[cache](https://github.com/actions/cache)

### 缓存文档

官方文档也有说明 [缓存文档](https://help.github.com/cn/actions/automating-your-workflow-with-github-actions/caching-dependencies-to-speed-up-workflows)

缓存大致原理就是把目标路径打包存储下来，并记录一个唯一key。

下次启动时，根据key去查找。找到了就再按路径解压开。

### 缓存大小限制

注意缓存有大小限制。对于免费用户，单个包不能超过500MB，整个仓库的缓存不能超过2G。

### 缓存运作流程

一般我们在任务步骤中增加一个cache

```
  steps:
    ...
    - use: actions/cache@v1
      with:
        ...
    ...
```

那么在这个地方，缓存执行的操作是restore。

在steps的末尾，会自动增加一个PostCache，执行的操作是record。

## Qt项目的缓存优化

Qt项目每次运行Actions时，都是先通过[install-qt-action](https://github.com/jurplel/install-qt-action)模板，安装Qt，之后再获取代码，编译运行。

安装Qt这个步骤，可快可慢，涛哥在windows平台测试下来，平均要1分30秒左右。

加上cache后，平均只有25秒。

### 无缓存的配置

先看一个Qt项目的编译配置


```yml
name: Windows
on: [push,pull_request]
jobs:
  build:
    name: Build
    runs-on: windows-latest
    strategy:
      matrix:
        qt_ver: [5.12.6]
        qt_target: [desktop]
        qt_arch: [win64_msvc2017_64, win32_msvc2017]
        include:
          - qt_arch: win64_msvc2017_64
            msvc_arch: x64
          - qt_arch: win32_msvc2017
            msvc_arch: x86
    # 步骤
    steps:
      # 安装Qt
      - name: Install Qt
        uses: jurplel/install-qt-action@v2.0.0
        with:
          version: ${{ matrix.qt_ver }}
          target: ${{ matrix.qt_target }}
          arch: ${{ matrix.qt_arch }}
      # 拉取代码
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1
      # 编译msvc
      - name: build-msvc
        shell: cmd
        env:
          vc_arch: ${{ matrix.msvc_arch }}
        run: |
          call "C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\VC\Auxiliary\Build\vcvarsall.bat" %vc_arch%
          qmake
          nmake
```

### 加缓存

缓存步骤，一般尽量写steps最前面。

```yml
    # 步骤
    steps:
      # 缓存
      - name: cacheQt
        id: WindowsCacheQt
        uses: actions/cache@v1
        with:
          path: ../Qt/${{matrix.qt_ver}}/${{matrix.qt_arch_install}}
          key: ${{ runner.os }}-Qt/${{matrix.qt_ver}}/${{matrix.qt_arch}}
```
install-qt-action有默认的Qt安装路径${RUNNER_WORKSPACE}，不过这个环境变量不一定能取到。

涛哥实际测试下来，以当前路径的上一级作为Qt路径即可。

### 环境变量还原

缓存只是把文件还原了，环境变量并没有还原，我们还需要手动还原环境变量。

install-qt-action这个模板增加了一个环境变量Qt5_Dir,值为Qt的安装路径,并把对应的bin添加到了Path。

我们要做的，就是在缓存恢复成功后，重新设置这两个变量，并去掉install-qt的步骤。

```yml
      - name: setupQt
        if: steps.WindowsCacheQt.outputs.cache-hit == 'true'
        shell: pwsh
        env:
          QtPath: ../Qt/${{matrix.qt_ver}}/${{matrix.qt_arch_install}}
        run: |
          $qt_Path=${env:QtPath}
          echo "::set-env name=Qt5_Dir::$qt_Path"
          echo "::add-path::$qt_Path/bin"   
```

steps.WindowsCacheQt.outputs.cache-hit == 'true'

是缓存模板的输出值，可以作为后续步骤的条件判断。

### 最终配置

写个伪配置，简单示例一下缓存流程

steps:
  - cache
  - setupQt
    if: cache-hit == 'true'
  - installQt
    if: cache-hit = 'false'


实际配置

```yml
name: Windows
on: 
  # push代码时触发workflow
  push:
    # 忽略README.md
    paths-ignore:
      - 'README.md'
      - 'LICENSE'
  # pull_request时触发workflow
  pull_request:
    # 忽略README.md
    paths-ignore:
      - 'README.md'
      - 'LICENSE'
jobs:
  build:
    name: Build
    # 运行平台， windows-latest目前是windows server 2019
    runs-on: windows-latest
    strategy:
      # 矩阵配置
      matrix:
        qt_ver: [5.12.6]
        qt_target: [desktop]
        # mingw用不了
        # qt_arch: [win64_msvc2017_64, win32_msvc2017, win32_mingw53,win32_mingw73]
        qt_arch: [win64_msvc2017_64, win32_msvc2017]
        # 额外设置msvc_arch
        include:
          - qt_arch: win64_msvc2017_64
            msvc_arch: x64
            qt_arch_install: msvc2017_64
          - qt_arch: win32_msvc2017
            msvc_arch: x86
            qt_arch_install: msvc2017
    env:
      targetName: HelloActions-Qt.exe
    # 步骤
    steps:
      - name: cacheQt
        id: WindowsCacheQt
        uses: actions/cache@v1
        with:
          path: ../Qt/${{matrix.qt_ver}}/${{matrix.qt_arch_install}}
          key: ${{ runner.os }}-Qt/${{matrix.qt_ver}}/${{matrix.qt_arch}}
      - name: setupQt
        if: steps.WindowsCacheQt.outputs.cache-hit == 'true'
        shell: pwsh
        env:
          QtPath: ../Qt/${{matrix.qt_ver}}/${{matrix.qt_arch_install}}
        run: |
          $qt_Path=${env:QtPath}
          echo "::set-env name=Qt5_Dir::$qt_Path"
          echo "::add-path::$qt_Path/bin"          
      # 安装Qt
      - name: Install Qt
        if: steps.WindowsCacheQt.outputs.cache-hit != 'true'
        # 使用外部action。这个action专门用来安装Qt
        uses: jurplel/install-qt-action@v2.0.0
        with:
          # Version of Qt to install
          version: ${{ matrix.qt_ver }}
          # Target platform for build
          target: ${{ matrix.qt_target }}
          # Architecture for Windows/Android
          arch: ${{ matrix.qt_arch }}
      # 拉取代码
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1
      # 编译msvc
      - name: build-msvc
        shell: cmd
        env:
          vc_arch: ${{ matrix.msvc_arch }}
        run: |
          call "C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\VC\Auxiliary\Build\vcvarsall.bat" %vc_arch%
          qmake
          nmake
```
