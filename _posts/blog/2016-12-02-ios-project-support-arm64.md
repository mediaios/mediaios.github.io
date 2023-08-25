---
layout: post
title: 让你的IOS工程支持arm64
description: 介绍如何让你的 ios 工程支持 arm64 ; 对 architectures 相关参数的说明 ; 解释 XCode 中的 project、target、scheme
category: blog
tag: iOS, Objective-C, arm64, armv7, armv7s
---
## 概述

这篇文章主要介绍在为 ios APP 设置支持 arm64 时候的相关参数的概念，以及对 XCode中的 project、target、scheme 概念的说明。介绍的顺序如下：

* ios app 为什么要支持 arm64
* 只支持arm64的app和只支持armv7的app在表现形式上行的区别
* XCode中设置 architectures 的各个参数的含义
* 让App支持32-bit和64-bit基本步骤
* XCode中的 project、scheme、target 的含义

## ios app 为什么要支持 arm64

苹果在2014年10月20号发布了一条消息：从明年的二月一号开始，提交到App Store的应用必须支持64-bit。详细消息地址为：[WWWDC2014 updates](https://developer.apple.com/news/?id=10202014a) 

从iPhone 5S的A7 CPU开始到刚刚发布的iPhone 6（A8 CPU）都已经支持64-bit ARM 架构。

* Xcode 5.0.1开始支持编译32-bit和64-bit的Binary
* 同时支持32-bit和64-bit，我们需要选择的minimum deployment target为 iOS 5.1.1
* 64-bit的Binary必须运行在支持64-bit的CPU上，并且最小的OS版本要求是 7.0.3

## 只支持arm64的app和只支持armv7的app在表现形式上行的区别

如果你的应用只支持arm64，那么如果在一个比较老的iphone(iphone4)上,你的应用很可能会经常Crash.

## Code中设置 architectures 的各个参数的含义

在 XCode 中的 `Build Setting` 中，可以看到 Architectures 组。下面这些参数我们需要理解：

* Architectures : 你想支持的指令集。（支持指令集是通过编译生成对应的二进制数据包实现的，如果支持的指令集数目有多个，就会编译出包含多个指令集代码的数据包，造成最终编译的包很大。）
* Build Active Architecture Only : 是否只编译当前设备适用的指令集（如果这个参数设为YES，使用iPhone 6调试，那么最终生成的一个支持ARM64指令集的Binary。一般在DEBUG模式下设为YES，RELEASE设为NO）
* Valid Architectures : 即将编译的指令集。（Valid architectures 和 Architecture两个集合的交集为最终编译生成的版本）

关于指令集的参考如下:[iOS Devices: Releases, Firmware, Instruction Sets, Screen Sizes](https://www.innerfence.com/howto/apple-ios-devices-dates-versions-instruction-sets)

## 让App支持32-bit和64-bit基本步骤

* 确保Xcode版本号>=5.0.1
* 更新project settings, minimum deployment target >= 5.1.1
* 改变Architectures为 Standard architectures（include 64-bit）
* 运行测试代码，解决编译warnings and errors，对照本文档或者官方文档 64-Bit Transition Guide for Cocoa Touch对相* 应地方做出修改。（编译器不能告诉我们一切）
* 在真实的64-bit机器上测试
* 使用Instruments查看内存使用问题

## XCode中的 project、scheme、target 的含义

### 现象说明

在开发中，如果搞不清楚这些概念，那么很可能会遇到编译参数设置错误，进而导致你的APP要么编译不过，要么在运行的时候经常发生Crash. 我就是因为参数设置出了问题，才导致 appstore 验证总是通过不了。下面我们详细介绍一些这个模块的内容。

### Project

Xcode中的 project里面包含了所有的源文件，资源文件和构建一个或者多个product的信息。project利用他们去编译我们所需的product，也帮我们组织它们之间的关系。一个project可以包含一个或者多个target。project定义了一些基本的编译设置，每个target都继承了project的默认设置，每个target可以通过重新设置target的编译选项来定义自己的特殊编译选项。

project包含了以下信息：

1. 源文件 :
	* 代码的头文件和实现文件
	* 静态库，动态库，
	* 资源文件（如文本，xml，plist等）
	* 图片资源
	* 界面资源文件（xib， storyboard等）
2. 在文件结构的导航中，采用group去组织文件（实际开发中，尽量使用实体文件夹）
3. project的编译级别配置文件如（debug， release）
4. target
5. 运行环境如：debug，test

project可以单独存在，或者存在于一个workspace中。

### target

target定义了构造一个product所需的文件和编译指令。一个target对应于一个product。target说白了就是
告诉编译系统要编译的文件和编译设置。编译指令就是根据`Build settings` and `Build phases`来确定的。

target之间可以进行依赖。如果一个target的编译需要另外一个target作为他的输入，那么我们就可以说前者依赖于后者。如果这两个target在同一个workspace里面，Xcode可以发现他们的依赖关系，这种依赖称之为隐式依赖。当然你可以通过设置，明确他们的依赖关系。

不同的target还可以定义完整的差异化的编译设置, 从简单的调整优化选项, 到增加条件编译所使用的编译条件, 以至于所使用的base SDK都可以差异化指定.

在`Build Phases` 中有如下选项需要注意： 

* Copy Bundle Resources 是指生成的product的.app内将包含哪些资源文件。通过Copy Bundle Resources中内容的不同设置, 我们可以让不同的product包含不同的资源, 包括程序的主图标等, 而不是把XCode的工程中列出的资源一股脑的包含进去.
* Compile Sources 是指将有哪些源代码被编译
* Link Binary With Libraries 是指编译过程中会引用哪些库文件

每个target可以使用一个独立, 不同的Info.plist文件. 

### scheme

scheme定义了编译集合中的若干target，编译时的一些设置以及要执行的测试集合。我们可以定义多个scheme，但是每次只能使用其中一个。我们可以设置scheme保存在project中还是workspace中。如果保存在project中，那么任意包含了这个工程的workspace都可以使用。如果保存在workspace中，那么只有这个workspace可以使用。

## 参考

* [iOS工程如何支持64-bit](http://chun.tips/blog/2014/10/21/iosgong-cheng-ru-he-zhi-chi-64-bit/)
* [Xcode中的 workspace, project, target, scheme](http://www.jianshu.com/p/1308a199f168)





