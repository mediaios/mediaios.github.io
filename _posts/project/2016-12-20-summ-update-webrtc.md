---
layout: post
title: 修改WebRTC源代码编译出我们需要的动态库(禁用webrtc声音采集进行单向接听)
description: 修改WebRTC源代码以适应Anywhere在视频直播的同时进行voip通话
category: project
tag: ios, webrtc, Swift,voip
---

## 概述

今天，这篇文章主要介绍如何修改 WebRTC 的源代码，来编译出满足我们需求的动态库。

### 引入问题

Anywhere是一款视频直播软件。WebRTC是提供实时多媒体通信的第三方库。我们的最新需求是要能在 Anywhere 直播期间进行voip通话。


#### 矛盾点分析

面对一个全新的需求，弄清技术本质，搞清楚哪些可行，哪些不可行，在开发中非常重要。下面我简单描述下技术矛盾：

* Anywhere在直播的同时，需要听到 R（可以理解为视频接收端） 端的实时声音，同时也需要采集声音。我们用的技术是 `AudioQueue`.
* WebRTC在进行语音通话的时,语音播放和录制用的是 `Audio Unit`. WebRTC用的 Audio Unit 比 AudioQueue更底层，所以当 WebRTC启动通话之后，Anywhere在Live的时候往R端传输音频信息就完全中断了。

通过以上分析，我们的出的结论就是：我们不能利用原来的技术来实现这个需求。

### 解决方法

通过以上描述，我们可以肯定，不可能让 `AudioQueue和` `Audio Unit` 同时工作，因为他们都会依赖于 `AVAudioSession` ,所以我们只能把其中一个的采集音频信息给去掉，只保留一个。可以利用这一个采集音频信息的源，再往底层传输。

#### 方法描述

下面介绍一下上述矛盾的解决方式：

* 修改需求为：当进行 voip通话的时候，我们只需要听到对方的声音，而不需要对方听到自己的声音。
* 修改 WebRTC 源代码，把它的声音采集部分去掉，直接拿到其需要播放的码流，然后把这个码流引入到我们的 `Audio Queue` 中，我们自己去播放。

#### 需要克服的问题

* 如何获取 WebRTC 的源码并进行编辑并打包成动态库
* 如何对动态库进行签名
* 从何处着手修改 WebRTC 的源代码以及注意事项

## 获取WebRTC源码并编译出动态库

### 把depot_tools添加到环境变量中

clone 完代码之后进入到 `webrtc_ios` 目录中，配置环境变量。（注意，每次打开一个新的terminal窗口时就需要配置一次）

	git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
	export PATH=`pwd`/depot_tools:"$PATH"

### 构建出 ios 动态库

当你修改完它的源代码之后，需要build出其动态库：

	cd src/webrtc/build/ios
	./build_ios_libs.sh

### 得到WebRTC.framework

当构建出动态库之后，我们需要看看它的庐山面目：

	cd src/out_ios_libs
	ls -lh WebRTC.framework

得到该动态库之后，就需要对其进行手动签名，不然你是不能使用该库的。

## 修改WebRTC源代码的整体思路以及关键点

### 修改思路

既然我们的音频信息不用传递给 WebRTC,那么我就把 WebRTC 的音频采集和播放都去掉，然后我们自己拿到它的音频流信息进行播放。

我们得到音频流，那就需要我们一层一层的封装最终来获取到音频流。

### 关键点

* 每个 audioBuffer的大小是 1024 个字节
* ``const size_t frames_per_buffer = playout_parameters_.frames_per_buffer();`` 得到的值是 512   ( 我是在 reset方法中设置的   frames_per_buffer_ = 512;)

## 最终打的patch

最终打的path文件存储在我的码云（ModifyWebRTC）