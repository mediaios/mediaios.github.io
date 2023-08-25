---
layout: post
title: AVAudioSession应用
description: 对工作中应用 AVAudioSession 的总结
category: blog
tag: iOS, Objective-C, avaudiosession
---

## 设置应用为 扬声/标准 输出模式

在开发中，特别是在开发voip应用的时候，需要设置扬声器和标准模式的切换。下面就分享一下原理和代码。

### 原理和步骤

1. 监听 ``AVAudioSessionRouteChangeNotification``通知，监听方法的名字为 (checkSpeakType)
2. 在checkSpeakType中，判断当前音频输出的类型，然后根据其类型进行切换。
	
### 对AVAudioSession中的类的方法的分析

```
[[AVAudioSession getInstance] currentRoute] // 获取当前线路 
```
上面方法的返回值类型是一个 ``AVAudioSessionRouteDescription``(线路描述类)， 这个类里面包含有两个属性:  

 * inputs: NSArray类型，数组里面的每个元素的类型都是 ``AVAudioSessionPortDescription``
 * outputs: NSArray类型，数组里面的每个元素的类型都是 ``AVAudioSessionPortDescription``

有此，我们可以看出，如果你在应用中要处理音频输入，则从 inputs 如数；如果要处理音频输出，则从 outputs 入手。

inputs 和outputs 数组中每个元素的类型都是 ``AVAudioSessionPortDescription`` 音频端口描述信息。下面我们具体分析一下这个类中包含的内容：

* portType
* portName
* UID
* channels `<AVAudioSessionChannelDescription>`
* dataSources `<AVAudioSessionDataSourceDescription>`
* selectDataSource `<AVAudioSessionDataSourceDescription>`
* preferredDataSource `<AVAudioSessionDataSourceDescription>`

如何判断当前应用是扬声器模式还是标准模式：

拿到 outputs 数组中的元素 <AVAudioSessionPortDescription> ,判断音频的 type 或 name 即可。

### 切换 标准/扬声器 模式的代码

	- (IBAction)onpressedbuttonSpeaker:(id)sender {
	    self.speakerBtn.tag = self.speakerBtn.tag == kSpeakerBtnTag_Normal ? kSpeakerBtnTag_Selected : kSpeakerBtnTag_Normal;
	    
	    AVAudioSessionPortOverride sessionPortOverride;
	    
	    switch (self.speakerBtn.tag) {
	        case kSpeakerBtnTag_Selected:
	        {
	            sessionPortOverride = AVAudioSessionPortOverrideSpeaker;
	        }
	            break;
	        case kSpeakerBtnTag_Normal:
	        {
	            sessionPortOverride = AVAudioSessionPortOverrideNone;
	        }
	            break;
	            
	        default:
	            break;
	    }
	    
	    AVAudioSession *audioSesson = [AVAudioSession sharedInstance];
	    
	    BOOL res = [audioSesson overrideOutputAudioPort:sessionPortOverride error:nil];
	    
	    if (res) {
	        NSLog(@"Setting speaker success");
	    } 
	}
 




