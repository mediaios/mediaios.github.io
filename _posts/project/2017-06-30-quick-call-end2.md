---
layout: post
title:  问题处理-频繁打挂电话2
description: 在Anywhere中的VOIP模块，快速重复的拨打电话和挂断电话的时候，有时候 app 会发生crash
category: project
tag: viop,webrtc
---

## 说明

上面一种解决方案没能解决频繁打挂电话引起的app崩溃的问题。 现在换一种方法处理。

## 解决方法

* 因为每一路电话的peerConn都需要做资源同步，所以我们利用GCD来管理共享的资源。
* 对于管理peerConn的容器，另外加锁来保持其资源同步
* 对于peerConn的创建时机做修改
* 当页面切换时，因为做了切换SDK操作，所以当电话拨出后，要等一段时间才能做页面切换


## 利用GCD同步资源

创建串行队列，把每一路电话对应的peerConn的操作，都放到串行队列中去执行。

## peerConnection的创建时机

现在的peerConn的创建逻辑不太合理。当前在如下情况下可能会创建peerConn:

* processCallRequest (信令服务器发来callrequest请求时)
* processCallResponse (信令服务器发来callresponse请求时)
* processOffer (信令服务器发来offer请求时)
* processAnswer (信令服务器发来answer时)
* processIce (信令服务器发来ICE时)

这种创建方式存在的问题：

我们发现，当用户进入TVUCC界面点击了打电话操作后，很快进入主页面（出发了挂电话操作），这时信令服务器还可能在返回诸如 offer,answer,ice的信令，此时挂断电话操作已经发生了,_pcFactory=nil了，不应该再去创建peerConn了。所以应该修改peerConn的创建时机。

新的peerConn的创建时机：

* processCallRequest (信令服务器发来callrequest请求时)    适用于接电话的case
* processCallResponse (信令服务器发来callresponse请求时)  适用于打电话的case


