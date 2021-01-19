---
title: 玩转Qt(11)-github-Actions自动化发行
photos: /images/QtActions2/2.png
tags:
  - Qt
  - 持续集成(CI)
categories: 玩转Qt
abbrlink: 26275
date: 2019-12-3 12:44:23
---
- [简介](#%e7%ae%80%e4%bb%8b)
- [Qt项目的编译流程](#qt%e9%a1%b9%e7%9b%ae%e7%9a%84%e7%bc%96%e8%af%91%e6%b5%81%e7%a8%8b)
- [Qt项目的发布流程](#qt%e9%a1%b9%e7%9b%ae%e7%9a%84%e5%8f%91%e5%b8%83%e6%b5%81%e7%a8%8b)
  - [查找依赖](#%e6%9f%a5%e6%89%be%e4%be%9d%e8%b5%96)
  - [制作包](#%e5%88%b6%e4%bd%9c%e5%8c%85)
  - [上传](#%e4%b8%8a%e4%bc%a0)
- [定制发布流程](#%e5%ae%9a%e5%88%b6%e5%8f%91%e5%b8%83%e6%b5%81%e7%a8%8b)
  - [发布时机](#%e5%8f%91%e5%b8%83%e6%97%b6%e6%9c%ba)
  - [打包步骤](#%e6%89%93%e5%8c%85%e6%ad%a5%e9%aa%a4)
  - [多平台发布](#%e5%a4%9a%e5%b9%b3%e5%8f%b0%e5%8f%91%e5%b8%83)
  - [最终配置](#%e6%9c%80%e7%bb%88%e9%85%8d%e7%bd%ae)
    - [windows版的最终配置](#windows%e7%89%88%e7%9a%84%e6%9c%80%e7%bb%88%e9%85%8d%e7%bd%ae)
    - [MacOS最终配置](#macos%e6%9c%80%e7%bb%88%e9%85%8d%e7%bd%ae)
  - [结果和代码](#%e7%bb%93%e6%9e%9c%e5%92%8c%e4%bb%a3%e7%a0%81)

## 简介

在上一篇文章《github-Actions自动化编译》中，介绍了github-Actions的基本用法，

本文来介绍github-Actions的自动化发布。

## Qt项目的编译流程

先来回顾一下,上一篇文章中的Qt项目的编译流程

1. 安装Qt环境
   
   这一步用第三方Action模板：[install-qt-action](https://github.com/jurplel/install-qt-action)

2. 获取项目代码
   
   这一步用Actions官方核心模板：[actions/checkout](https://github.com/actions/checkout)

3. 执行qmake、make

    这一步用自定义脚本，也可以换成cmake、qbs、gn、ninja等构建工具

4. 执行test

    这一步可以引入单元测试、自动化UI测试等。暂无完善的方案，以后再说。

5. 发布

    见下文。

## Qt项目的发布流程

Qt程序在编译完成后，发布的大致流程是：

1、 查找依赖库

2、制作压缩包或者安装包

3、上传压缩包或者安装包到网站、网盘。

### 查找依赖

Qt官方提供的查找依赖库的命令行工具，包括：Windows平台的Windeployqt、MacOS平台的Macosdeployqt。

在这两个平台，只使用Qt库的情况下，这两个工具足够了。

### 制作包

做压缩包比较简单。(我们常说的‘绿色软件’，就是一个压缩包)

一般安装7z、rar之类的压缩工具，用一条命令行就行了。

涛哥这里再说一下，github-Actions给所有平台都提供了PowerShell，而PowerShell内置了压缩命令Compress-Archive。

使用也很简单，只要路径和名字，例如：

```powershell
Compress-Archive -Path .\MyFolder 'MyRelease.zip'
```

做安装包，Qt官方有功能很全面的安装包制作工具：QtInstallFrameWork, 稍微翻看一下文档或者例子即可。本文先不展开了。

### 上传

github 本身提供了'Release'功能，每个仓库都有一个'Release'页面

![](/images/QtActions2/1.png)

可以将打包好的压缩包或者安装包，直接上传上去, 供他人下载。

![](/images/QtActions2/2.png)

github-Actions还提供了 创建'Release'、上传'Release'的模板

[actions/create-release](https://github.com/actions/create-release)

[actions/upload-release-asset](https://github.com/actions/upload-release-asset)

这两个模板的用法也很简单，在yml文件中直接use就行了，不赘述了。

## 定制发布流程

前面介绍了一些简单的理论，接下来通过实例，教大家github-Actions的使用。

以[HelloActions-Qt](https://github.com/jaredtao/HelloActions-Qt)项目为例，做一些定制。

需求如下：

1、每次提交代码，同时在Windows、MacOS、Ubuntu、Android、IOS五个平台编译

2、每次提交tag，在windows和MacOS平台制作软件包，并发布到同一个github-'Release'

需求1已经实现了，着重讨论一下需求2：

### 发布时机

'每次提交tag'限定了发布的时机。

涛哥尝试了一番，最终得到答案。

回顾一下, Windows平台的编译配置：

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
steps中的每一个步骤，可以有触发条件。我们可以在这里指定，只有github的事件为tag时才执行:

```yml
  steps:
    。。。
    # tag 打包
    - name: package
      if: startsWith(github.event.ref, 'refs/tags/')
      run: |
        。。。
```
### 打包步骤

这里给出一个实际的打包步骤：

```yml
      # tag 打包
      - name: package
        if: startsWith(github.event.ref, 'refs/tags/')
        env:
          VCINSTALLDIR: 'C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\VC'
          archiveName: ${{ matrix.qt_ver }}-${{ matrix.qt_target }}-${{ matrix.qt_arch }}
          targetName: HelloActions-Qt.exe
        shell: pwsh
        run: |
          # 创建文件夹
          New-Item -ItemType Directory ${env:archiveName}
          # 拷贝exe
          Copy-Item bin\${env:targetName} ${env:archiveName}\
          # 拷贝依赖
          windeployqt --qmldir . ${env:archiveName}\${env:targetName}
          # 打包zip
          Compress-Archive -Path ${env:archiveName} ${env:archiveName}'.zip'
          # 记录环境变量packageName给后续step
          $name = ${env:archiveName}
          echo "::set-env name=packageName::$name"
          # 打印环境变量packageName
          Write-Host 'packageName:'${env:packageName}
```
做一些说明：

* vs运行时

其中的VCINSTALLDIR环境变量，是给windeployqt用的。有了这个环境变量，windeployqt会去msvc的安装路径提取‘运行时安装程序’。

* 记录包名称

打包完以后，将包名设置为环境变量，后续的步骤就可以通过环境变量拿到包名字了。

普通的设置环境变量，在步骤执行完成后就失效了，

这里使用github-Actions的‘记录命令’set-env ，具体可以参考文档[github-Actions记录命令](https://help.github.com/zh/actions/automating-your-workflow-with-github-actions/development-tools-for-github-actions)         

文档说不要用双引号，应该都是针对linux的，我试出来的PowerShell用法如下：

```powershell
          $name = ${env:archiveName}
          echo "::set-env name=packageName::$name"
```

先取环境变量到一个局部变量，再在‘记录命令’中引用局部变量。

### 多平台发布

如果只有一个平台、一种配置，直接用那两个模板就能解决问题。

这是官方给的例子[upload-release-asset](https://github.com/actions/upload-release-asset)：

```yml
    steps:
      - name: Checkout code
        uses: actions/checkout@master
      - name: Build project # This would actually build your project, using zip for an example artifact
        run: |
          zip --junk-paths my-artifact README.md
      - name: Create Release
        id: create_release
        uses: actions/create-release@v1.0.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: false
      - name: Upload Release Asset
        id: upload-release-asset 
        uses: actions/upload-release-asset@v1.0.1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }} # This pulls from the CREATE RELEASE step above, referencing it's ID to get its outputs object, which include a `upload_url`. See this blog post for more info: https://jasonet.co/posts/new-features-of-github-actions/#passing-data-to-future-steps 
          asset_path: ./my-artifact.zip
          asset_name: my-artifact.zip
          asset_content_type: application/zip
```

在多平台 或者 多配置的情况下，同一个tag, 只有第一个执行create-release的任务可以成功，后续任务

再次执行create-release时，该tag下已经有了同名的‘Release’，所以会create失败。

这个问题折磨了涛哥好一阵子。找不到现成的解决方案，涛哥就自己实现了一种:

1. 先用github的REST API去判断该tag下有没有‘Release’:
  
    没有则执行create-release，并提取upload_url；
    
    有则提取upload_url。
    
2. 最后执行upload-release-asset 
   

调用REST API，涛哥依旧使用了方便的PowerShell,

实际的配置如下：

```yml
      # tag 查询github-Release
      - name: queryReleaseWin
        id: queryReleaseWin
        if: startsWith(github.event.ref, 'refs/tags/')
        shell: pwsh
        env:
          githubFullName: ${{ github.event.repository.full_name }}
          ref: ${{ github.event.ref }}
        run: |
          [string]$tag = ${env:ref}.Substring(${env:ref}.LastIndexOf('/') + 1)
          [string]$url = 'https://api.github.com/repos/' + ${env:githubFullName} + '/releases/tags/' + ${tag}
          $response={}
          try {
            $response = Invoke-RestMethod -Uri $url -Method Get
          } catch {
            Write-Host "StatusCode:" $_.Exception.Response.StatusCode.value__ 
            Write-Host "StatusDescription:" $_.Exception.Response.StatusDescription
            # 没查到，输出
            echo "::set-output name=needCreateRelease::true"  
            return
          }
          [string]$latestUpUrl = $response.upload_url
          Write-Host 'latestUpUrl:'$latestUpUrl
          if ($latestUpUrl.Length -eq 0) {
            # 没查到，输出
            echo "::set-output name=needCreateRelease::true"  
          }
      # tag 创建github-Release
      - name: createReleaseWin
        id: createReleaseWin
        if: startsWith(github.event.ref, 'refs/tags/') && steps.queryReleaseWin.outputs.needCreateRelease == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        uses: actions/create-release@v1.0.0
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body: ${{ github.event.head_commit.message }}
          draft: false
          prerelease: false
      # tag 重定向upload_url到环境变量uploadUrl。
      - name: getLatestTagRelease
        if: startsWith(github.event.ref, 'refs/tags/')
        shell: pwsh
        env:
          githubFullName: ${{ github.event.repository.full_name }}
          upUrl: ${{ steps.createReleaseWin.outputs.upload_url }}
          ref: ${{ github.event.ref }}
        run: |
          # upUrl不为空，导出就完事
          if (${env:upUrl}.Length -gt 0) {
              $v=${env:upUrl}
              echo "::set-env name=uploadUrl::$v"
              return
          } 
          # upUrl为空则重新获取
          [string]$tag = ${env:ref}.Substring(${env:ref}.LastIndexOf('/') + 1)
          [string]$url = 'https://api.github.com/repos/' + ${env:githubFullName} + '/releases/tags/' + ${tag}
          $response = Invoke-RestMethod -Uri $url -Method Get
          [string]$latestUpUrl = $response.upload_url
          Write-Host 'latestUpUrl:'$latestUpUrl
          # 导出
          echo "::set-env name=uploadUrl::$latestUpUrl"
          Write-Host 'env uploadUrl:'${env:uploadUrl}
      # tag 上传Release
      - name: uploadRelease
        id: uploadRelease
        if: startsWith(github.event.ref, 'refs/tags/')
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        uses: actions/upload-release-asset@v1.0.1
        with:
          upload_url: ${{ env.uploadUrl }}
          asset_path: ./${{ env.packageName }}.zip
          asset_name: ${{ env.packageName }}.zip
          asset_content_type: application/zip
```
### 最终配置

#### windows版的最终配置

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
      # 安装Qt
      - name: Install Qt
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
      # tag 打包
      - name: package
        if: startsWith(github.event.ref, 'refs/tags/')
        env:
          VCINSTALLDIR: 'C:\Program Files (x86)\Microsoft Visual Studio\2019\Enterprise\VC'
          archiveName: ${{ matrix.qt_ver }}-${{ matrix.qt_target }}-${{ matrix.qt_arch }}
        shell: pwsh
        run: |
          # 创建文件夹
          New-Item -ItemType Directory ${env:archiveName}
          # 拷贝exe
          Copy-Item bin\${env:targetName} ${env:archiveName}\
          # 拷贝依赖
          windeployqt --qmldir . ${env:archiveName}\${env:targetName}
          # 打包zip
          Compress-Archive -Path ${env:archiveName} ${env:archiveName}'.zip'
          # 记录环境变量packageName给后续step
          $name = ${env:archiveName}
          echo "::set-env name=packageName::$name"
          # 打印环境变量packageName
          Write-Host 'packageName:'${env:packageName}
      # tag 查询github-Release
      - name: queryReleaseWin
        id: queryReleaseWin
        if: startsWith(github.event.ref, 'refs/tags/')
        shell: pwsh
        env:
          githubFullName: ${{ github.event.repository.full_name }}
          ref: ${{ github.event.ref }}
        run: |
          [string]$tag = ${env:ref}.Substring(${env:ref}.LastIndexOf('/') + 1)
          [string]$url = 'https://api.github.com/repos/' + ${env:githubFullName} + '/releases/tags/' + ${tag}
          $response={}
          try {
            $response = Invoke-RestMethod -Uri $url -Method Get
          } catch {
            Write-Host "StatusCode:" $_.Exception.Response.StatusCode.value__ 
            Write-Host "StatusDescription:" $_.Exception.Response.StatusDescription
            # 没查到，输出
            echo "::set-output name=needCreateRelease::true"  
            return
          }
          [string]$latestUpUrl = $response.upload_url
          Write-Host 'latestUpUrl:'$latestUpUrl
          if ($latestUpUrl.Length -eq 0) {
            # 没查到，输出
            echo "::set-output name=needCreateRelease::true"  
          }
      # tag 创建github-Release
      - name: createReleaseWin
        id: createReleaseWin
        if: startsWith(github.event.ref, 'refs/tags/') && steps.queryReleaseWin.outputs.needCreateRelease == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        uses: actions/create-release@v1.0.0
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body: ${{ github.event.head_commit.message }}
          draft: false
          prerelease: false
      # 重定向upload_url到环境变量uploadUrl。
      - name: getLatestTagRelease
        # tag 上一步无论成功还是失败都执行
        if: startsWith(github.event.ref, 'refs/tags/')
        shell: pwsh
        env:
          githubFullName: ${{ github.event.repository.full_name }}
          upUrl: ${{ steps.createReleaseWin.outputs.upload_url }}
          ref: ${{ github.event.ref }}
        run: |
          # upUrl不为空，导出就完事
          if (${env:upUrl}.Length -gt 0) {
              $v=${env:upUrl}
              echo "::set-env name=uploadUrl::$v"
              return
          } 
          [string]$tag = ${env:ref}.Substring(${env:ref}.LastIndexOf('/') + 1)
          [string]$url = 'https://api.github.com/repos/' + ${env:githubFullName} + '/releases/tags/' + ${tag}
          $response = Invoke-RestMethod -Uri $url -Method Get
          [string]$latestUpUrl = $response.upload_url
          Write-Host 'latestUpUrl:'$latestUpUrl
          echo "::set-env name=uploadUrl::$latestUpUrl"
          Write-Host 'env uploadUrl:'${env:uploadUrl}
      # tag 上传Release
      - name: uploadRelease
        id: uploadRelease
        if: startsWith(github.event.ref, 'refs/tags/')
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        uses: actions/upload-release-asset@v1.0.1
        with:
          upload_url: ${{ env.uploadUrl }}
          asset_path: ./${{ env.packageName }}.zip
          asset_name: ${{ env.packageName }}.zip
          asset_content_type: application/zip
```

#### MacOS最终配置

```yml
name: MacOS
on: 
  push:
    paths-ignore:
      - 'README.md'
      - 'LICENSE'
  pull_request:
    paths-ignore:
      - 'README.md'
      - 'LICENSE'
jobs:
  build:
    name: Build
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [macos-latest]
        qt_ver: [5.12.6]
        qt_arch: [clang_64]
    env:
      targetName: HelloActions-Qt
    steps:
      - name: Install Qt
        uses: jurplel/install-qt-action@v2.0.0
        with:
          version: ${{ matrix.qt_ver }}
      - uses: actions/checkout@v1
        with:
          fetch-depth: 1
      - name: build macos
        run: |
          qmake
          make
      # tag 打包
      - name: package
        if: startsWith(github.event.ref, 'refs/tags/')
        run: |
          # 拷贝依赖
          macdeployqt bin/${targetName}.app -qmldir=. -verbose=1 -dmg
      # tag 查询github-Release
      - name: queryRelease
        id: queryReleaseMacos
        if: startsWith(github.event.ref, 'refs/tags/')
        shell: pwsh
        env:
          githubFullName: ${{ github.event.repository.full_name }}
          ref: ${{ github.event.ref }}
        run: |
          [string]$tag = ${env:ref}.Substring(${env:ref}.LastIndexOf('/') + 1)
          [string]$url = 'https://api.github.com/repos/' + ${env:githubFullName} + '/releases/tags/' + ${tag}
          $response={}
          try {
            $response = Invoke-RestMethod -Uri $url -Method Get
          } catch {
            Write-Host "StatusCode:" $_.Exception.Response.StatusCode.value__ 
            Write-Host "StatusDescription:" $_.Exception.Response.StatusDescription
            # 没查到，输出
            echo "::set-output name=needCreateRelease::true"  
            return
          }
          [string]$latestUpUrl = $response.upload_url
          Write-Host 'latestUpUrl:'$latestUpUrl
          if ($latestUpUrl.Length -eq 0) {
            # 没查到，输出
            echo "::set-output name=needCreateRelease::true"  
          }
      # tag 创建github-Release
      - name: createReleaseWin
        id: createReleaseWin
        if: startsWith(github.event.ref, 'refs/tags/') && steps.queryReleaseMacos.outputs.needCreateRelease == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        uses: actions/create-release@v1.0.0
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          body: ${{ github.event.head_commit.message }}
          draft: false
          prerelease: false
      # 重定向upload_url到环境变量uploadUrl。
      - name: getLatestTagRelease
        # tag 上一步无论成功还是失败都执行
        if: startsWith(github.event.ref, 'refs/tags/')
        shell: pwsh
        env:
          githubFullName: ${{ github.event.repository.full_name }}
          upUrl: ${{ steps.queryReleaseMacos.outputs.upload_url }}
          ref: ${{ github.event.ref }}
        run: |
          # upUrl不为空，导出就完事
          if (${env:upUrl}.Length -gt 0) {
              $v=${env:upUrl}
              echo "::set-env name=uploadUrl::$v"
              return
          } 
          [string]$tag = ${env:ref}.Substring(${env:ref}.LastIndexOf('/') + 1)
          [string]$url = 'https://api.github.com/repos/' + ${env:githubFullName} + '/releases/tags/' + ${tag}
          $response = Invoke-RestMethod -Uri $url -Method Get
          [string]$latestUpUrl = $response.upload_url
          Write-Host 'latestUpUrl:'$latestUpUrl
          echo "::set-env name=uploadUrl::$latestUpUrl"
          Write-Host 'env uploadUrl:'${env:uploadUrl}
      # tag 上传Release
      - name: uploadRelease
        id: uploadRelease
        if: startsWith(github.event.ref, 'refs/tags/')
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        uses: actions/upload-release-asset@v1.0.1
        with:
          upload_url: ${{ env.uploadUrl }}
          asset_path: ./bin/${{ env.targetName }}.dmg
          asset_name: ${{ env.targetName }}.dmg
          asset_content_type: application/applefile
```

### 结果和代码

![](/images/QtActions2/2.png)

代码在github [HelloActions-Qt](https://github.com/jaredtao/HelloActions-Qt)

另外在涛哥的Qml控件库TaoQuick，也使用了这一套配置

 [TaoQuick](https://github.com/jaredtao/TaoQuick)
