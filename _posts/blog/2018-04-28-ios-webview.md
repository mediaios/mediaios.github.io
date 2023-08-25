---
layout: post
title: UIWebView和WKWebView的使用以及注意点
description: ios平台嵌入网页是很常见的一个操作，那么UIWebView和WKWebView两种加载网页的方式到底怎么样呀呢？有哪些坑呢？本文将详细介绍之。
category: blog
tag: ios
---

## UIWebView和WKWebView的比较

UIWebView是一个很老的web容器，从`ios2.0`开始。它有很多的弊端，比如：加载速度慢，占用内存多，优化困难等。如果加载网页过多，还可能因为过量占用内存而给系统kill掉。各种优化的方法效果也不那么明显。

下面我们用UIWebView和WKWebView分别每十秒加载一个网页，我们来比较一下它们的内存使用情况。

`UIWebView`内存使用：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180428IosWebView/uiwebview_01.png)

`WKWebView`内存使用：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180428IosWebView/wkwebview_01.png)

可见`WKWebView`确实比`UIWebView`使用了更少的内存。并且在运行的过程中，我很明显的感受到了`WKWebView`对网页的加载速度要快于`UIWebView` 。

