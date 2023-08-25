---
layout: post
title: AVFoundation 工作总结
description: 对 AVFoundation 的开发工作的细节整理
category: blog
tag: ios, Objective-C, Swift
---

## 1.设置分辨率


### 知识点说明

当我们为 AVCaptureSession 的 sessionPreset属性就是分辨率属性，在分辨率值中有一个属性需要我们充分注意 `AVCaptureSessionPresetInputPriority` .

该属性表示 the capture session does not control audio and video output settings ，也就是说当前的AVCaptureSession不是你当前正在使用的（因为它当前没有控制音视频输出设置）。所以说你的应用里很可能有多个 AVCaptureSession 对象。

所以，当你对其中一个 AVCaptureSession 对象设置一些设备级别的参数时，很可能会影响另一个 AVCaptureSession 对象。因为这些参数不是某一对象所拥有的，它们的掌控权不是对象级别。 这些参数目前知道的有： 分辨率、对焦类型（自动和手动）。



### 工作中遇到的场景

TVUAnywhere 是一款视频直播应用。它有两个页面开启了capture,分别是主页面和扫描二维码页面。下面是其具体参数设置：

* MainPage: 支持自动对焦和手动对焦，分辨率支持360、480、720、1080P等。
* ScanQRCodePage: 没有设置对焦模式和分辨率。

我们的应用出现了下面两种异常：

* 当我们为主页面的capture设置为低分辨率时，二维码扫描的成功率就大大降低。
* 我们在主页面对着远景对焦(触摸一下屏幕之后，对焦方式就变成了手动对焦)，然后在进入二维码扫描界面，此时看到的二维码非常模糊。当然，扫描的成功率也是大大降低。



### 调试及解决

面对以上现象，首先我觉得两个问题很可能是同一种原因引起的，那就是MainPage对capture参数的设置影响到了 ScanQRCodePage 的capture.然后我就在一个每秒刷一次的timer中打印这两个capture 的分辨率（sessionPreset）。通过调试我发现（对焦方式的调试可以使用KVO）：

 * 应用在 MainPage 中时，二维码页面的分辨率是 `AVCaptureSessionPresetInputPriority`
 * 应用在 ScanQRCodePage 中时，主页面的分辨率是 `AVCaptureSessionPresetInputPriority`
 * 两个页面的分辨率设置和对焦方式设置彼此互相影响。

最后，我把二维码页面的 capture 分辨率设置死为 1080P ，对焦方式为自动对焦。当再次回到主页面时，设置主页面分辨率为原来的值。


### 代码


#### 分辨率

为设置分辨率而做的简单封装，具体代码如下：

```
/**
 *  Reset resolution
 *
 *  @param m_session     AVCaptureSession instance
 *  @param g_height_size
 */
+ (void)resetSessionPreset:(AVCaptureSession *)m_session andHeight:(int)g_height_size
{
    [m_session beginConfiguration];
    switch (g_height_size) {
        case 1080:
            m_session.sessionPreset = [m_session canSetSessionPreset:AVCaptureSessionPreset1920x1080] ? AVCaptureSessionPreset1920x1080 : AVCaptureSessionPresetHigh;
            break;
        case 720:
            m_session.sessionPreset = [m_session canSetSessionPreset:AVCaptureSessionPreset1280x720] ? AVCaptureSessionPreset1280x720 : AVCaptureSessionPresetMedium;
            break;
        case 480:
            m_session.sessionPreset = [m_session canSetSessionPreset:AVCaptureSessionPreset640x480] ? AVCaptureSessionPreset640x480 : AVCaptureSessionPresetMedium;
            break;
        case 360:
            m_session.sessionPreset = AVCaptureSessionPresetMedium;
            break;
            
        default:
            break;
    }
    [m_session commitConfiguration];
}

```


#### 对焦模式

相机的对焦模式有两种，一种是自动对焦，另一种是手动对焦。

```
// 设置对焦模式
  self.QRDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
  
 [self.QRDevice lockForConfiguration:&error];
    [self.QRDevice setValue:[NSNumber numberWithInteger:AVCaptureFocusModeContinuousAutoFocus] forKey:@"focusMode"];  // 利用KVO的方式
    [self.QRDevice setFocusMode:AVCaptureFocusModeContinuousAutoFocus]; 
    [self.QRDevice unlockForConfiguration];

```