---
layout: post
title: OC与JS交互
description: 介绍OC如何与JS进行交互：js中调用OC代码，OC调用JS代码
category: blog
tag: OC,js,OC与JS交互,UIWebView and JS
---

## 概述

本文主要介绍OC与JS之间如何进行交互。具体包括两方面的内容：

* OC中调用JS代码
* JS调用OC

## 在OC中调用JS代码

在OC中调用JS代码应该有多种形式，最常见的是调用js中的函数，获取JS函数中的变量,获取JS函数的返回值等。

### 在OC中调用JS函数

首先晒出我的html代码：

	<head>
	   <meta charset="UTF-8"/>
	   <title>hello world</title>
	   <script type="text/javascript">
		function showTitle()
		{
		 alert(document.title);
		}
		function getTitle()
		{
			return document.title;
		}
		</script>
	<style> 
	.div-height{border:1px solid #F00; height:200px} 
	.button_qi{width:100;height:100; background: green;margin-left: 100}
	</style> 
	</head>
	<div class="div-height"> </div>
	<body bgcolor='red'>
	
	</body>

上面的js代码只是简单的写了一个显示网页title的函数，用的是alert弹框的形式。

我们在OC中只需要利用下面的代码便能调用js中的showTitle函数：

	[webView stringByEvaluatingJavaScriptFromString:@"showTitle();"]; // webView是UIWebView的实例

### 在OC中获取JS函数的返回值

同样是利用UIWebView的 `stringByEvaluatingJavaScriptFromString:` 方法，其返回值就是对应的js函数的返回值。

### 在OC中获取JS变量

未完待续……

## 在JS中调用OC方法

若要实现在 JS中调用OC的方法，那么必须要满足下面条件：

* 每次发送请求前都会调用
* 利用该方法作为JS和OC之间的桥梁
* 在JS跳转网页
* 在OC代理方法中通过判断自定义协议头, 决定是否是JS调用OC方法
* 在OC代理方法中通过截取字符串, 获取JS想调用的OC方法名称

### 较简单类型

首先晒出我的html代码：

	<head>
      <meta charset="UTF-8"/>
      <script type="text/javascript">
       function fn_click() {
	         // 自定义协议，进行动态跳转
          window.location.href = 'dz://click';
        }

		</script>
	<style> 
	.div-height{border:1px solid #F00; height:200px} 
	.button_qi{width:100;height:100; background: green;margin-left: 100}
	</style> 
	</head>
	<div class="div-height"> </div>
	<body bgcolor='red'>
	    <div>
	        <button  class = "button_qi" onclick="fn_click();">按钮1</button>
	    </div>
	   <div>	
	</body>

上面html代码的意思就是在页面中点击 ‘按钮1’ 会触发`fn_click()`函数，js函数中自定义了一个协议进行动态跳转。 `dz://click`  dz就是自定义的协议，然后我们可以在OC中截取这一段字符串，然后在OC中决定走哪个方法。


然后在 UIWebViewDelegate的方法中做如下：

	- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
	{
	    NSString *urlStr = request.URL.absoluteString;
	    // 判断请求是否是自定义的，如果是自定义的，则进行处理
	    NSRange range = [urlStr rangeOfString:@"dz://"];
	    NSUInteger loc = range.location;
	    if (loc != NSNotFound) {
	        NSString *method = [urlStr substringFromIndex:loc + range.length]; // 取出JS方法的方法名
	        NSLog(@"qizhang----debug-----%@",method);
	        self.tipLabel.hidden = NO;
	        self.tipLabel.text = [NSString stringWithFormat:@"JS 函数为：%@",method];
	        
	        SEL sel = NSSelectorFromString(method); // 将JS方法名转化为OC的方法名
	        [self performSelector:sel withObject:nil];
	
	    }
	    return YES;
	}
	
	- (void)click {
    NSLog(@"点击了btn");
	}

### 较复杂类型

html代码：

	<head>
    <meta charset="UTF-8"/>
    <title>hello world</title>
    <script type="text/javascript">

	function load(){
	 document.getElementById("target").onclick();
	}	
	function repost()
	{
	 location.href = "http://www.520it.com";
	}
	function sum()
	{
	 return 1+1;
	}
	function btnClick()
	{
	  location.href = "xmg://sendMessageWithNumber_andContent_?10086";
	}
	</script>
	<style> 
	.div-height{border:1px solid #F00; height:200px} 
	.button_qi{width:100;height:100; background: green;margin-left: 100}
	</style> 
	</head>
	<div class="div-height"> </div>
	<body bgcolor='red' onload="load()">
	   <button style = "background: red; height: 150px; width: 150px;" onclick = "btnClick();">the button</button>
	
	</body>
	
然后在 UIWebViewDelegate的方法中做如下：

	- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType
	{
	    NSString *schem = @"xmg://";
	    NSString *urlStr = request.URL.absoluteString;
	    if ([urlStr hasPrefix:schem]) {
	        // 1.从URL中获取方法名称
	        // xmg://sendMessageWithNumber_andContent_?10086&love
	        NSString *subPath = [urlStr substringFromIndex:schem.length];
	         // 注意: 如果指定的用于切割的字符串不存在, 那么就会返回整个字符串
	        NSArray *subPaths = [subPath componentsSeparatedByString:@"?"];
	        // 2.获取方法名称
	         NSString *methodName = [subPaths firstObject];
	        methodName = [methodName stringByReplacingOccurrencesOfString:@"_" withString:@":"];
	        NSLog(@"%@", methodName);
	        // 2.调用方法
	        SEL sel = NSSelectorFromString(methodName);
	        // 3.处理参数
	                 NSString *parma = nil;
	                 if (subPaths.count == 2) {
	                    parma = [subPaths lastObject];
	                    // 3.截取参数
	                    NSArray *parmas = [parma componentsSeparatedByString:@"&"];
	                    [self performSelector:sel withObject:[parmas firstObject] withObject:[parmas lastObject]];
	                     return NO;
	                  }
         //这里处理参数很麻烦，我们可以自己实现一个类去将苹果自带的方法进行优化，其实就是可以传递不同的参数（多个）去处理相应的事件
                 [self performSelector:sel withObject:parma];
                 return NO;
             }
        return YES;
	}
	
	- (void)sendMessageWithNumber:(NSString *)number andContent:(NSString *)content status:(NSString *)status
	{
	     NSLog(@"发信息给%@, 内容是%@, 发送的状态是%@", number, content, status);
	}
	
## 代码下载

* [github下载（OC_JS/WebViewTest/UIWebViewDemo/）](https://github.com/MaxwellQi/OC_JS)


	