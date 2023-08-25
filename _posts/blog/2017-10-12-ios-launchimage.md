---
layout: post
title: LaunchImage的设置及对应图片尺寸
description: 对 App LaunchImage设置所需要的不同图片尺寸说明
category: blog
tag: icons,ios
---

## 设置APP的LaunchImage

按照如下步骤设置app的LaunchImage:

`In Assets.xcassets click + button -> App Icons & Launch Images -> New iOS Launch Image`


在`xcode9`中，添加LaunchImage之后如图：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171012LaunchImage/launchiamge_01.png)

## LaunchImage所需图片尺寸信息

下面罗列出launchImage所需图片的所有尺寸信息。

### iPhoneX Portrait ios 11+

* iphone X : 1125 x 2436

### iPhoneX Landscape ios 11+

* iphone X : 2436 x 1125

### iPhone Portrait ios 8,9

* Retina HD 5.5 : 1242 x 2208 (iphone 6 plus)
* Retina HD 4.7 : 750 x 1334  (iphone 6)

### iPhone Landscape ios8,9

* Retina HD 5.5 : 2208 x 1242 (iphone 6 plus)

### iPhone Portrait ios 7-9

* 2x : 640 x 960  (iphone 4)
* Retina 4 : 640 x 1136 (iphone 5)

### iPad Portrait ios7-9

* 1x : 768 x 1024
* 2x : 1536 x 2048

### iPad Landscape ios7-9

* 1x : 1024 x 768
* 2x : 2048 x 1536

### iPhone Portrait ios 5,6

* 1x : 320 x 480 (iphone 3)
* 2x : 640 x 960 (iphone 4)
* Retina 4 : 640 x 1136 (iphone 5)

### iPad Portrait Without Status Bar ios 5,6

* 1x : 768 x 1004
* 2x : 1536 x 2008

### iPad Portrait ios 5,6

* 1x : 768 x 1024
* 2x : 1536 x 2048

### iPad Landscape With Status Bar ios 5,6

* 1x : 1024 x 748
* 2x : 2048 x 1946

### iPad Landscape ios 5,6

* 1x : 1024 x 768 
* 2x : 2048 x 1536


综上，如图所示：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171012LaunchImage/launchiamge_02.png)


## 参考

* [Launch Screen](https://developer.apple.com/ios/human-interface-guidelines/icons-and-images/launch-screen/)
* [iOS: Launch Image for all devices, include iPad Pro](https://stackoverflow.com/questions/39900225/ios-launch-image-for-all-devices-include-ipad-pro)
* [IOS launch images - driving me crazy](https://stackoverflow.com/questions/34112681/ios-launch-images-driving-me-crazy)
* [LaunchImage(s) Explained](https://medium.com/@hellosunschein/launchimage-s-explained-33b88c02d027)



