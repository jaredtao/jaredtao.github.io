---
title: Qt项目持续集成系列之-github自动化编译
photos: /img/avatar.jpg
tags:
  - Qt
  - 持续集成(CI)
categories: Qt进阶之路
abbrlink: 26275
date: 2019-04-30 12:44:23
---

## 简介

持续集成的概念和好处，涛哥就不再赘述了。

本文的目标是，领各位读者入门，学会如何在GitHub上搭建Qt项目的自动化编译环境。

后续的文章还会有Qt项目的自动化测试、及多平台自动发行。

## 创建一个Qt工程


这里使用默认的HelloWorld模板。

![](/images/QtCI/1.png)

文件结构如下：

![](/images/QtCI/2.png)

## 为代码创建git仓库

使用命令行操作

git init

git add .

git commit –a –m “init version”

或者使用小乌龟(tortoiseGit)

![](/images/QtCI/3.png)

## 在github上创建仓库

![](/images/QtCI/4.png)

## 上传代码到github

使用命令行

git push https://github.com/yourpath/xxxxxxxxxxx master

或者小乌龟

![](/images/QtCI/5.png)
![](/images/QtCI/6.png)

## 使用Travis

travis是一个第三方的CI网站，提供linux和osx的docker环境，可以与github集成。

使用这个网站的docker，只需要在代码git仓库中放一个叫”.travis.yml”的配置文件即可，文件具体内容在下文中。

网址： https://travis-ci.org/

如果想知道更多关于travis的内容，访问帮助文档网址：https://docs.travis-ci.com/

需要先注册/登陆。默认用github账号就好了。
![](/images/QtCI/7.png)

第一次使用会提示github账号认证之类的，通过就行了。

登陆成功后，进入仓库管理界面，点击那个加号

![](/images/QtCI/8.png)

进入添加仓库界面，找到要添加的仓库HelloCI，打开开关。如果列表中没有仓库，可以点击左上角的Sync account进行同步，之后再去找仓库。

![](/images/QtCI/9.png)

![](/images/QtCI/10.png)

## 使用appveyor

appveyor是一个第三方的CI网站，提供windows的docker环境，可以与github集成。

使用这个网站的docker，只需要提供一个叫” appveyor.yml”的配置文件即可，文件具体内容在下文中。

网址：https://www.appveyor.com/

帮助文档网址：https://www.appveyor.com/docs/

![](/images/QtCI/11.png)

登陆界面。这个网站用Github账号不一定能正常登陆，请自行尝试。

本人使用的是微软的Visual Studio Team 账号

![](/images/QtCI/12.png)

登陆成功后，进入项目列表。点击上面的New Project进入添加项目页面。

![](/images/QtCI/13.png)

选github标签，然后找到HelloCI，点击ADD按钮

![](/images/QtCI/14.png)

添加成功了

![](/images/QtCI/15.png)

## 添加CI配置文件

.travis.yml （后文有链接，可下载到）
![](/images/QtCI/16.png)


travis默认系统为ubuntu，并提供一些基础的命令。但是没有安装Qt，这里通过ubuntu源进行安装，选择的版本为5.9.6。

关于ubuntu源 在这个网站上查看细节 https://launchpad.net/~beineri/+archive/ubuntu/


在搜索框输入想要的qt版本，查看是否有对应的源、如何使用


appveyor.yml

![](/images/QtCI/17.png)

Appveyor比较方便一些，已经装好了各种版本的vs 和Qt，这里使用vs 14.0和qt 5.9.5 msvc2015_64

其它版本详情看这里https://www.appveyor.com/docs/windows-images-software/

后续的自动化测试、覆盖率统计、自动部署也是在这两个配置文件里实现，这次先不说了。

配置文件添加到仓库

![](/images/QtCI/18.png)

通常为了方便查看CI状态，我们会写一个README.md的文件，里面链接上CI仓库和仓库对应的状态图标（专业名字叫徽章/badge）

![](/images/QtCI/19.png)
上图四个红框依次编号1-4，那么

1是travis的小图标链接，在travis网站上，对应仓库里

![](/images/QtCI/20.png)

点击那个小图标，在弹出的页面中，选择代码分支为master，格式为MardDown
![](/images/QtCI/21.png)

（也可以仿照我提供的格式来写，我的那个是个表格的方式，看着更好一些）

2是travis仓库对应的链接

3是appveyor的状态图标，在appveyor的对应仓库中找到：

![](/images/QtCI/22.png)
4是appveyor的对应仓库链接

添加好之后的文件结构
![](/images/QtCI/23.png)


提交修改到github，触发CI

用命令行

git add .

git commit –a –m “add CI and README”

git push xxx master

或者小乌龟

![](/images/QtCI/24.png)
![](/images/QtCI/25.png)
![](/images/QtCI/26.png)

提交好了，到github上看一看吧
![](/images/QtCI/27.png)

这次提交已经触发了CI。

上图这四个按钮分别 对应前面的四个链接，可以点开查看状态。

状态图标里面显示的状态是默认的，等一段时间后(CI运行完成)，刷新即可看到最新的状态。

也可以在github 仓库的commit栏，点击CI状态按钮，来查看CI信息。

![](/images/QtCI/28.png)

## github仓库链接

https://github.com/jaredtao/HelloCI

