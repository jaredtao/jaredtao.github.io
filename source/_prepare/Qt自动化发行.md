---
title: Qt项目持续集成系列之-github自动化发行
photos: /img/avatar.jpg
tags:
  - Qt
  - 持续集成(CI)
categories: Qt进阶之路
abbrlink: 40291
date: 2019-04-30 14:44:23
---

## 简介

本文的目标是，在github上实现Qt工程的自动化发行。

看个预览图先:
![s](/images/QtCI/28.png)

上图所示github的Release中，包含了两个macos平台的dmg包、5个windows平台的zip包以及一个ubuntu平台的包，都是自动化发行的结果。

后续会加入Android和ios包。

不懂github持续集成的读者，请先去看上一篇文章

https://jaredtao.github.io/2019/04/30/Qt%E8%87%AA%E5%8A%A8%E5%8C%96%E7%BC%96%E8%AF%91/

接下来将要介绍Appveyor和Travis的配置

## Appveyor的配置

上一篇文章中已经介绍过，Appveyor网站提供了windows的Docker镜像，包含各种版本的visual studio和Qt，这里通过yml配置文件，来使用这些镜像。

先是通过一个矩阵来配置不同的镜像
```yml
environment:
  matrix:
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2015
    platform: x86
    qt: 5.9
    releaseName: HelloCI_qt59_vs2015_x86
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2015
    platform: x64
    qt: 5.9
    releaseName: HelloCI_qt59_vs2015_x64
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2017
    platform: x64
    qt: 5.9
    releaseName: HelloCI_qt59_vs2017_x64
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2015
    platform: x64
    qt: 5.12
    releaseName: HelloCI_qt512_vs2015_x64
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2017
    platform: x64
    qt: 5.12
    releaseName: HelloCI_qt512_vs2017_x64
```
matrix之下，每一个‘-’号开头的一段描述，都代表一个镜像的配置。
其中APPVEYOR_BUILD_WORKER_IMAGE是网站提供的环境变量，用来指定Docker镜像。
platform、 qt以及releaseName是我自定义的变量，用来在后文中执行脚本时做区分
```yml
before_build:
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2015" set msvc=msvc2015
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2015" set vs=C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2017" set msvc=msvc2017
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2017" set vs=C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Auxiliary\Build
  - if "%platform%"=="x86" set QTDIR=C:\Qt\%qt%\%msvc%
  - if "%platform%"=="x64" set QTDIR=C:\Qt\%qt%\%msvc%_64
  - set PATH=%PATH%;%QTDIR%\bin;
  - if "%platform%"=="x86" set vcvarsall=%vs%\vcvarsall.bat
  - if "%platform%"=="x64" set vcvarsall=%vs%\vcvarsall.bat
  - if "%platform%"=="x86" call "%vcvarsall%" x86
  - if "%platform%"=="x64" call "%vcvarsall%" x64
```
before_build脚本，就是编译之前要执行的脚本，默认使用windows的cmd语法，当然也可以使用Powershell等。
我这段脚本目的是，在编译之前，先确认要使用的vs编译器和Qt所在路径。
vs一般通过调用vcvarsall.bat脚本，并传入x86或x64等参数，来确定相应的编译器和链接器，如cl.exe、nmake.exe等。
Qt的使用，是将QTDIR\bin设置到环境变量PATH中，以便可以找到qmake.exe。

```yml
build_script:
  - qmake
  - nmake
```
有了正确的环境设置，接下来就是编译了，很简单的qmake make

```yml
after_build:
  - if "%APPVEYOR_REPO_TAG%"=="true" windeployqt bin\HelloCI.exe --qmldir %QTDIR%\qml

```
编译通过后，有一个after_build的脚本执行时机，此时的环境变量和前面一样。
我这段脚本目的是，在代码被打上tag的情况下，执行windeployqt命令，将依赖的Qt库都打包到一起。
APPVEYOR_REPO_TAG是Docker提供的环境变量，有tag的时候这个变量的值为true。

```yml
artifacts:
  - path: bin
    name: $(releaseName)
```
artifacts用来描述可以发行的代码包，这里指定了bin路径，则Docker会自动将bin路径打包为zip，zip的名字是指定的name。

```yml
deploy:
  provider: GitHub
  auth_token: $(GITHUB_OAUTH_TOKEN)
  description: 'HelloCI Release'
  draft: false
  prerelease: false
  on:
      APPVEYOR_REPO_TAG: true
```
deploy用来描述要发行的目标网站。
 这里的provider指定为github，就可以使用github的Release功能了。当然还有很多其它的网站也可用来发行。
 auth_token是从github上生成的验证用的token，后面再说如何生成。为了不让别人看到具体token，这里引用了自定义的环境变量GITHUB_OAUTH_TOKEN

```yml
     on： 
     	APPVEYOR_REPO_TAG: true
```
 表示只有在代码被打上tag的时候，才执行deploy。
 这里的deploy没有指定artifacts，默认会把定义的artifacts都发行出去，这就够了。


最终的yml配置文件

appveyor.yml
```yml
version: '{build}'

branches:
  except:
    - project/travis
environment:
  matrix:
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2015
    platform: x86
    qt: 5.9
    releaseName: HelloCI_qt59_vs2015_x86
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2015
    platform: x64
    qt: 5.9
    releaseName: HelloCI_qt59_vs2015_x64
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2017
    platform: x64
    qt: 5.9
    releaseName: HelloCI_qt59_vs2017_x64
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2015
    platform: x64
    qt: 5.12
    releaseName: HelloCI_qt512_vs2015_x64
  - APPVEYOR_BUILD_WORKER_IMAGE: Visual Studio 2017
    platform: x64
    qt: 5.12
    releaseName: HelloCI_qt512_vs2017_x64
before_build:
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2015" set msvc=msvc2015
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2015" set vs=C:\Program Files (x86)\Microsoft Visual Studio 14.0\VC
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2017" set msvc=msvc2017
  - if "%APPVEYOR_BUILD_WORKER_IMAGE%"=="Visual Studio 2017" set vs=C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Auxiliary\Build
  - if "%platform%"=="x86" set QTDIR=C:\Qt\%qt%\%msvc%
  - if "%platform%"=="x64" set QTDIR=C:\Qt\%qt%\%msvc%_64
  - set PATH=%PATH%;%QTDIR%\bin;
  - if "%platform%"=="x86" set vcvarsall=%vs%\vcvarsall.bat
  - if "%platform%"=="x64" set vcvarsall=%vs%\vcvarsall.bat
  - if "%platform%"=="x86" call "%vcvarsall%" x86
  - if "%platform%"=="x64" call "%vcvarsall%" x64
build_script:
  - qmake
  - nmake
after_build:
  - if "%APPVEYOR_REPO_TAG%"=="true" windeployqt bin\HelloCI.exe --qmldir %QTDIR%\qml
artifacts:
  - path: bin
    name: $(releaseName)
deploy:
  provider: GitHub
  auth_token: $(GITHUB_OAUTH_TOKEN)
  description: 'HelloCI Release'
  draft: false
  prerelease: false
  on:
      APPVEYOR_REPO_TAG: true
```
## Travis

Travis网站有两个，https://travis-ci.org 和https://travis-ci.com， 区别是后者可以操作github的私有仓库。

来看看travis的配置

```yml
matrix:
  include:
    - os: linux
      dist: xenial
      env: 
        targetFile=HelloCI
        releaseName=HelloCI_ubuntu_xenial_x64
      cache:
        bundler: true
        apt: true
        directories:
            - /opt/qt512/
    - os: osx
      osx_image: xcode10.2
      env: 
        targetFile=HelloCI
        releaseName=HelloCI_macos10-14_xcode10-2.dmg
      cache:
        bundler: true
        directories:
            - /usr/local/Cellar/qt/5.12.2/
    - os: osx
      osx_image: xcode9.4
      env: 
        targetFile=HelloCI
        releaseName=HelloCI_macos10-13_xcode9-4.dmg
      cache:
        bundler: true
        directories:
            - /usr/local/Cellar/qt/5.12.2/
```
其中
```yml
  os: linux
    dist: xenial
```
这是ubuntu16.04的配置（目前还不支持18.04 bionic）
env中自定义了一些环境变量，后面用到
cache是用来缓存的，这里指定了Qt的安装路径，那么在下一次同样的Docker镜像启动时，优先使用缓存，除非没有安装或者有新版本才会去更新这个路径。这样可以节省大量的Docker运行时间哦。
```yml
    os: osx
      osx_image: xcode10.2
```      
这是macos系统的设置，osx_image是travis提供的，可在其文档中找到其它可用版本
https://docs.travis-ci.com/user/reference/osx/
缓存也是要有的。
  

osx系统通过brew安装Qt，只需要一条命令即可 brew install qt，按照brew官网`https://brew.sh/`的描述，目前安装的版本为5.12.2
涛哥试过后，发现安装路径是/usr/local/Cellar/qt/5.12.2/

ubuntu系统安装Qt，上一篇文章也说过了，是通过launchpad源，目前已经有了最新的5.12.2和5.12.3了。

最终的travis.yml
```yml
#dist: xenial
#指定语言为cpp
language: cpp
#需要sudu权限
sudo: required
#编译器为gcc
compiler: gcc
#环境变量
#env: QT_BASE="512"

matrix:
  include:
    - os: linux
      dist: xenial
      env: 
        targetFile=HelloCI
        releaseName=HelloCI_ubuntu_xenial_x64
      cache:
        bundler: true
        apt: true
        directories:
            - /opt/qt512/
    - os: osx
      osx_image: xcode10.2
      env: 
        targetFile=HelloCI
        releaseName=HelloCI_macos10-14_xcode10-2.dmg
      cache:
        bundler: true
        directories:
            - /usr/local/Cellar/qt/5.12.2/
    - os: osx
      osx_image: xcode9.4
      env: 
        targetFile=HelloCI
        releaseName=HelloCI_macos10-13_xcode9-4.dmg
      cache:
        bundler: true
        directories:
            - /usr/local/Cellar/qt/5.12.2/
  
#缓存

# 注意上面的缓存,指定qt安装路径，可以避免重复安装

#组
group: deprecated-2019Q1

# travis默认系统为ubuntu，并提供一些基础的命令。但是没有安装Qt，需要通过ubuntu源进行安装。
# 关于ubuntu源 在这个网站上查看细节 https://launchpad.net/~beineri/+archive/ubuntu/
# 当然也可以通过qt安装包 +一些命令的方式来安装，这里以源的方式为主。
#安装前的设置
#添加qt5.12.1的源
before_install:
    - if [[ "$TRAVIS_OS_NAME" == "osx" ]]; then brew update; brew install qt; fi 
    - if [[ "$TRAVIS_OS_NAME" == "linux" ]]; then sudo add-apt-repository ppa:beineri/opt-qt-5.12.3-xenial -y; sudo apt-get update -qq; sudo apt-get install -y libglew-dev libglfw3-dev; sudo apt-get install -y qt512-meta-minimal; fi
# 单独安装qt3d模块的示例
#    - sudo apt-get install -y qt5123d

before_script:
    - if [[ "$TRAVIS_OS_NAME" == "linux" ]]; then source /opt/qt512/bin/qt512-env.sh; fi
script:
    - if [[ "$TRAVIS_OS_NAME" == "linux" ]]; then source /opt/qt512/bin/qt512-env.sh; qmake ; make ; fi
    - if [[ "$TRAVIS_OS_NAME" == "osx" ]]; then /usr/local/Cellar/qt/5.12.2/bin/qmake ; make ; fi
before_deploy:
    - if [[ "$TRAVIS_OS_NAME" == "osx" ]]; then /usr/local/Cellar/qt/5.12.2/bin/macdeployqt bin/${targetFile}.app -qmldir=/usr/local/Cellar/qt/5.12.2/qml -verbose=1 -dmg ; mv bin/${targetFile}.dmg bin/${releaseName} ; fi
    - if [[ "$TRAVIS_OS_NAME" == "linux" ]]; then mv bin/${targetFile} bin/${releaseName} ; fi
deploy:              # 部署
  provider: releases # 部署到 GitHub Release，除此之外，Travis CI 还支持发布到 fir.im、AWS、Google App Engine 等
  api_key: $GITHUB_OAUTH_TOKEN # 填写 GitHub 的 token （Settings -> Personal access tokens -> Generate new token）
  file: bin/${releaseName}   # 部署文件路径
  skip_cleanup: true     # 设置为 true 以跳过清理,不然 apk 文件就会被清理
  on:     # 发布时机           
    tags: true       # tags 设置为 true 表示只有在有 tag 的情况下才部署   
notifications:
    email: false


```
## OAuth token

OAuth机制，是在不知道密码的情况下，只通过授权的token去访问特定的api。

使用token大致步骤：
![s](/images/QtCI/29.png)
![s](/images/QtCI/30.png)
![s](/images/QtCI/31.png)
![s](/images/QtCI/32.png)
![s](/images/QtCI/33.png)
![s](/images/QtCI/34.png)

接下来要去两个地方粘贴，最好先贴到记事本或着什么地方，备用。

先到Travis的项目配置页面
![s](/images/QtCI/35.png)
![s](/images/QtCI/36.png)

Name写前面的yml配置文件中引用的变量名GITHUB_OAUTH_TOKEN,value就是前面的token

接下来是Appveyor
![s](/images/QtCI/37.png)
![s](/images/QtCI/38.png)

一样的东西，填好后保存即可。

最后，给代码打上tag，提交吧。

涛哥在连续提交了50多次后。。。
![s](/images/QtCI/39.png)
最终拿到了这两个徽章
![s](/images/QtCI/40.png)
还有这正确的发布包。
（ubuntu那个，缺少linuxdeployqt打包步骤，后面再加）
![s](/images/QtCI/28.png)

当然，你们不用像我一样提交50多次，你们可以直接用我写好的配置文件。

最后附上GitHub链接
https://github.com/jaredtao/HelloCI