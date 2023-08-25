---
layout: post
title:  第三方集成索引
description: 对 IOS 开发中所用到的第三方库集成的总结
category:
tag: 
---

## VOIP 通信

### Socket.io的集成
	
- [OC版本的集成][2]
- [C++版本的集成][3]

### WebRTC的集成

- [初期版本--基于apprtc-ios(libjingle)集成][4]
- [最终版本--利用最新的webrtc库完成语音通话][5]

### 对voip通信的总结

- [WebRTC通信的原理][6]
- [完成voip通信的历程及注意事项][7]
- [Integration WebRTC with TVUAnywhere][8]
- [修改WebRTC源代码编译出我们需要的动态库(禁用webrtc声音采集进行单向接听)][9]
- [修改WebRTC源代码编译出我们需要的动态库(替换webrtc声音采集进行双向通话)][15]
- [Anywhere一对多通话设计][10]

## DJI OSMO

### 示例程序

- [Preview大疆设备的视频到手机上][11]

### 文档总结

- [Anywhere集成OSMO开发设计][12]
- [Anywhere集成OSMO的开发过程][14]
- [视频帧的时间戳][13]




[MaxwellQi]: https://maxwellqi.github.io "MaxwellQi"
[1]: {{ page.url }} ({{ page.title }})
[2]: https://github.com/MaxwellQi/Socket.io_Objective-C
[3]: https://github.com/MaxwellQi/Socket.io_CPP
[4]: https://github.com/MaxwellQi/WebRTC_IOS
[5]: https://github.com/MaxwellQi/WebRTC_IOS_final
[6]: https://maxwellqi.github.io/summ-webrtc-knowledge
[7]: https://maxwellqi.github.io/summ-voip-warning
[8]: https://maxwellqi.github.io/summ-integration-webrtc
[9]: https://maxwellqi.github.io/summ-update-webrtc
[10]: https://maxwellqi.github.io/one-to-many-webrtc
[11]: https://github.com/MaxwellQi/OSMO_DJI
[12]: https://maxwellqi.github.io/summ-integration-osmo
[13]: https://maxwellqi.github.io/summ-osmo-frame-time-stamp
[14]: https://maxwellqi.github.io/summ-dev-osmo
[15]: https://maxwellqi.github.io/summ-update-webrtc02