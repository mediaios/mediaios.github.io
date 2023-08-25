---
layout: post
title: AVFoundation中的相机操作
description: AVFoundation中的相机操作，包括设置分辨率、自动聚焦、放大缩小聚焦、反转摄像头、拍照、音频录制和播放、视频录制和播放；讲解使用AVFoundation需要的注意事项等。
category: blog
tag: ios, Objective-C, Swift, AVFoundation
---

## 关于相机操作

AVFoundation主要是处理多媒体的框架

### 简要对AVFoundation说明

……



## 相机操作的例子

### 放大缩小capture

```
// 配合着捏合手势
#define kPrinchVelocityDividerFactor 20.0f
#define kMaxZoomFactor 5.0f
+ (void)zoomCapture:(UIPinchGestureRecognizer *)recognizer
{
    AVCaptureDevice *videoDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    [videoDevice formats];
    if ([recognizer state] == UIGestureRecognizerStateChanged) {
        NSError *error = nil;
        if ([videoDevice lockForConfiguration:&error]) {
            CGFloat desiredZoomFactor = videoDevice.videoZoomFactor + atan2f(recognizer.velocity, kPrinchVelocityDividerFactor);
            videoDevice.videoZoomFactor = desiredZoomFactor <= kMaxZoomFactor ? MAX(1.0, MIN(desiredZoomFactor, videoDevice.activeFormat.videoMaxZoomFactor)) : kMaxZoomFactor ;
            [videoDevice unlockForConfiguration];
        } else {
            NSLog(@"error: %@", error);
        }
    }

}
```

### 设置分辨率

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

### 设置帧率

```
+ (void)settingFrameRate:(int)frameRate
{
    AVCaptureDevice *captureDevice = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    [captureDevice lockForConfiguration:NULL];
    [captureDevice setActiveVideoMinFrameDuration:CMTimeMake(1, frameRate)];
    [captureDevice unlockForConfiguration];
}
```

### 设置曝光

曝光分为自动曝光和手动曝光。下面是具体代码：

	AVCaptureDevice *device = video_input.device;
	        NSError *error;
	        if ([device lockForConfiguration:&error]) {
	            [device setExposurePointOfInterest:self.clickPoint];
	            [device setExposureMode:AVCaptureExposureModeCustom];
	            
	            [device setExposureModeCustomWithDuration:CMTimeMake(1, 20) ISO:value completionHandler:^(CMTime syncTime) {
	                NSLog(@"set success...");
	            }];
	            [device unlockForConfiguration];
	         }

我们可以通过 `activeFormat.minISO` 和 `activeFormat.maxISO` 来分别获取最小曝光值和最大曝光值，我们所设置的曝光值就在这两个值之间。
