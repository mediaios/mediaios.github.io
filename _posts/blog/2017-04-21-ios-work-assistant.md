---
layout: post
title: ios开发编程辅助
description: 介绍工作中经常用到的一些方法
category: blog
tag: ios
---

## 获取手机连接的wifi热点名称

工作中我们经常遇到判断手机连接的是哪个热点，下面是具体代码：

	#import <SystemConfiguration/CaptiveNetwork.h>
	
	CFArrayRef myArray = CNCopySupportedInterfaces();
	CFDictionaryRef myDict = CNCopyCurrentNetworkInfo(CFArrayGetValueAtIndex(myArray, 0));
	//    NSLog(@"SSID: %@",CFDictionaryGetValue(myDict, kCNNetworkInfoKeySSID));
	NSString *networkName = CFDictionaryGetValue(myDict, kCNNetworkInfoKeySSID);
	
	if ([networkName isEqualToString:@"Hot Dog"])
	{
	    self.storeNameController = [[StoreDataController alloc] init];
	    [self.storeNameController addStoreNamesObject];
	}
	else {
	    UIAlertView *alert = [[UIAlertView alloc]initWithTitle: @"Connection Failed"
	                                                   message: @"Please connect to the Hot Dog network and try again"
	                                                  delegate: self
	                                         cancelButtonTitle: @"Close"
	                                         otherButtonTitles: nil];
	
	    [alert show];