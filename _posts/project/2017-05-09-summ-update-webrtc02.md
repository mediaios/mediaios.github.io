---
layout: post
title: 修改WebRTC源代码编译出我们需要的动态库(替换webrtc声音采集进行双向通话)
description: 修改WebRTC源代码以适应Anywhere在视频直播的同时进行voip通话
category: project
tag: ios, webrtc, Swift,voip
---

## 概述

在上一篇文章中，我们介绍了如何通过修改`webrtc`的源代码来实现单向的voip通话，那么在这篇文章中，我们将介绍如何修改`webrtc`的源代码来实现双向通话。

## 引入问题

Anywhere是一款视频直播软件。WebRTC是提供实时多媒体通信的第三方库。我们的最新需求是要能在 Anywhere 直播期间进行voip通话。

### 矛盾点分析

矛盾点分析在上一篇文章中就已经总结过了，我们在此不过多描述。

### 解决方法

我们知道，在上一篇文章中，我们是将webrtc的音频采集模块去掉，只播放另一端的音频，因此对方听不到我们这一端的声音。这不是真正意义上的voip通话，通话必须是双向的。所以我们为了解决能进行双向通话的问题，就自然想到能不能利用`anywhere`中的 `Audio Queue`采集到的音频信息替换掉`webrtc`采集的音频信息呢？答案是肯定的。 我们只需要查看一下`webrtc`源代码，找到`webrtc`采集的音频信息传递给了底层的哪个方法，哪个变量，然后我们就用`Anywhere中的Audio queue`替换掉`webrtc`的音频采集。这样就能完成webrtc的双向通话了。

## 在上层回调中的调用

在`Anywhere`中的 `Audio queue`的call back中的调用：


		static void inputBufferHandler(void *                                 inUserData,
	                               AudioQueueRef                          inAQ,
	                               AudioQueueBufferRef                    inBuffer,
	                               const AudioTimeStamp *                 inStartTime,
	                               UInt32                                 inNumPackets,
	                               const AudioStreamPacketDescription*	  inPacketDesc) {
	
	    TVUWebRTCManager *webRTCManager = [TVUWebRTCManager shareInstance];
	    if (webRTCManager) {
	        if ([webRTCManager voipIsCalling]) {
	            RTCPeerConnection *rtcPeerConn = [[TVUWebRTCManager shareInstance] getNewestPeerConnection];
	            [rtcPeerConn acceptRecodedAudioData:(int8_t *)inBuffer->mAudioData size:inBuffer->mAudioDataByteSize];
	        }
	    }
	  }
	

## 最终打的patch

最终打的patch文件存储在我的码云(ModifyWebRTC/double_talk)