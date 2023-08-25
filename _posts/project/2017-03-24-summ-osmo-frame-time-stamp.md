---
layout: post
title: 如何构造每一帧的时间戳
description: 介绍什么是视频的时间戳以及如何构造每一帧的时间戳
category: project
tag: ios, osmo, h264
---

## 关于每一帧的时间戳

每一帧的数据都包含时间戳信息。在进行视频的编码和解码的时候，都需要为每一帧添加时间戳信息。时间戳的值一般为当你获取到当前帧时的当前时间。这样说可能有些抽象，下面我列举两种情况：

* 当你需要进行视频编码时，你获取到YUV数据之后，在进行编码之前需要把每一帧的时间戳给设置上去，此时这个时间戳可以取当前host时间。
* 当你从网络上得到视频流进行解码的时候，在解码之前你需要把每一帧的时间戳给设置上去，此时这个时间戳可以取当前host时间。

如果我们的应用有固定的帧率(FPS）那么，相邻的两帧的时间戳的差值应该是很近似于一个固定值。那么我们可以根据帧率这一特性，来自己构造每一帧的时间戳，具体方法视情况而定。我们在下面会介绍两种时间戳的构造方法。

## 如果一次parser出来很多帧数据（时间戳一样）如何构造新的时间戳

如果仅仅只看标题，很难让人理解这是什么意思。我们下面来看一下具体的场景。

### 场景

Anywhere继承了大疆OSMO的功能，具体是：当Anywhere发现有OSMO热点时，会自动连接上该热点，并获取OSMO的视频数据显示在Anywhere上。OSMO的视频数据也能被传到Transport层中进行live.

我们在开发中发现，DJI的接收视频流的代理方法，每一次可能会包含多帧数据，也就是利用ffmpeg进行parse时，有时候能parse出多帧数据，那么就需要我们如何为这parse出来多帧的每一帧构造一个合理的时间戳。

### 原理图

![](https://raw.githubusercontent.com/MaxwellQi/MaxwellQi.github.io/a15431134e3a5df3b256e042e432487f391f2e54/images/project/summ-osmo-frame-time-stamp/time-stamp_01.png)

说明：<font color="red">上图是按照FPS=30来进行画的</font>

### 代码

java版

	private  int    [ ]  interval_array = new int[4];
	private static final int INTERVAL_FRAME = 33  1000  1000;
	private static final int MAX_INTERVAL_FRAME = 250  1000  1000;
	private static final int MIN_INTERVAL_FRAME = 15  1000  1000;
	private  long  time_pre = 0;//pre timestamp
	private  int     index = 0;
	private  long time_current_modified = 0;
	private  long time_former_modified = 0;
	
	private tvuRawFrame modifyFrameTimestamp(tvuRawFrame data) {
			int len = 0;
			int sum = 0;
			long time_current = data.timestamp;
			int average_interval;
			len = interval_array.length;

		if (time_current - time_pre > MAX_INTERVAL_FRAME) {
			Log.e(TAG, "reset timestamp when frame interval exceed 250ms");
			time_pre = time_current;
			time_former_modified = time_current - INTERVAL_FRAME;
			for (int i = 0; i < len; i++) {
				interval_array[i] = INTERVAL_FRAME;
			}
		}
		index = index % len;
		interval_array[index] = Long.valueOf(time_current - time_pre).intValue();

		/*compute average interval*/
		for (int i = 0; i < len; i++) {
			sum += interval_array[i];
		}

		average_interval = sum / len;
		time_current_modified = time_former_modified + Integer.valueOf(average_interval).longValue();
		if (average_interval > (INTERVAL_FRAME + MIN_INTERVAL_FRAME) || average_interval < (INTERVAL_FRAME - MIN_INTERVAL_FRAME))
			Log.d(TAG, "curT: " + time_current + " laT:" + time_pre +
					" diff:" + (time_current - time_pre) / 1000000 + " MT:" + time_current_modified +
					" ltMT:" + time_former_modified + " interval:" + (time_current_modified - time_former_modified) / 1000000 +
					" offset:" + (time_current_modified - time_current) / 1000000);

		time_pre = time_current;
		time_former_modified = time_current_modified;
		interval_array[index] = average_interval;
		data.timestamp = time_current_modified / 1000;
		index ++;

		return data;
}

OC版：
	
	// _interval_array = [NSMutableArray arrayWithArray:@[@0.0f,@0.0f,@0.0f,@0.0f,@0.0f,@0.0f]];
	static Float64 time_pre = 0; // per timestamp
	static Float64 time_current_modified = 0;
	static Float64 time_former_modified = 0;
	static Float64 interval_frame = 0.033;
	static int temp_index = 0;
	- (Float64)modifyFrameTimestamp:(Float64)value
	{
    
    Float64 time_current = value;
    Float64 average_interval;
    int len = 0;
    Float64 sum = 0;
    len = (int)_interval_array.count;
    
    if (time_current - time_pre > 0.25 ) {
        log4cplus_error("DJIOSMO","reset timestamp when frame interval exceed 250ms");
        time_pre = time_current;
        time_former_modified = time_current - interval_frame;
        for (int i = 0 ;i < len; i++) {
            self.interval_array[i] = [NSNumber numberWithFloat:interval_frame];
        }
    }
    temp_index = temp_index % len;
    _interval_array[temp_index] =  [NSNumber numberWithFloat:(time_current - time_pre)];
    
    // 计算平均值
    for (int i = 0; i < len; i++) {
        sum += [_interval_array[i] floatValue];
    }
    average_interval = sum / len;
    time_current_modified = time_former_modified + average_interval;
    
    log4cplus_debug("DJIOSMO", "decord before:curT:%f , laT:%f , diff:%f , MT:%f , ltMT:%f , interval:%f , offset:%f ",
          time_current,
          time_pre,
          (time_current - time_pre),
          time_current_modified,
          time_former_modified,
          (time_current_modified-time_former_modified),
          (time_current_modified - time_current)
          );
    
    
    time_pre = time_current;
    time_former_modified = time_current_modified;
    _interval_array[temp_index] = [NSNumber numberWithFloat:average_interval];
    temp_index ++;
    return time_current_modified;;
}




## 如何根据hosttime自己构造时间戳

hosttime是一个时间（可能是从上次重启设备到现在的时间，也可能是其它的一个具体时间）。我们可以根据host time和FPS来构造时间。具体原理为：

	第一帧的时间戳 = hostTime
	下一帧的时间戳 = 上一帧的时间戳 + 33ms

下面是我在解码之前，为每一帧构造的时间戳：

	const CMTime interval = CMTimeMake(1, 30);
        static CMTime cur_ts;
        if(initOSMOTime == 0){
            CMClockRef hostClockRef = CMClockGetHostTimeClock();
            CMTime hostTime = CMClockGetTime(hostClockRef);
            Float64 curtime = CMTimeGetSeconds(hostTime);
            curtime = curtime - 0.25;
            cur_ts = CMTimeMakeWithSeconds(curtime, 30);
            initOSMOTime = 1;
        }else{
            cur_ts = CMTimeAdd(cur_ts, interval);
            Float64 modified_curtime = CMTimeGetSeconds(cur_ts);
            
            
            
            curtime = [self getCurrentTimestamp];
            curtime = curtime - 0.25;
            
            Float64 cur_interval = curtime - modified_curtime;
            if(cur_interval >=0.015 ){ //one frame interval 0.033s
                cur_ts = CMTimeMakeWithSeconds(curtime, 30);
                log4cplus_error("DJIOSMO", "decode>0.033 reset base time, because current interval:%f exceed [-0.066,0.066]", cur_interval);
            }
        }
        
        CMSampleTimingInfo timingInfo;
        timingInfo.duration = kCMTimeInvalid;
        timingInfo.decodeTimeStamp = cur_ts;
        timingInfo.presentationTimeStamp = cur_ts;

在解码的时候，把`timingInfo`传给解码器。

## 构造时间戳的原理

主要是第一帧如何赋值。然后后面的帧可以根据FPS赋值，也可以根据其它合理的方式赋值。


