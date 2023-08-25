---
layout: post
title:  接收 DJI inspire1 码流并解码
description: 对Anywhere接收 inspire1 的码流进行解码时中途失败原因的分析; 应用激活以及大疆无人机绑定
category: project
tag: DJI,DJI inspire1,inspire1,HWDecoder
---

## 场景说明

在做`inspire1`视频流解码时，遇到了以下场景。

在解码大疆设备的码流时，我们的解码器在解码`OSMO`的视频流时完全没有问题，可以一直解码成功。但是当视频流源换为`DJI inspire1`的时候，解码器总是在解一段时间突然失败，报`-12909`错误，即使重启解码器也不能解决这种问题。

在进行解码的过程中，有可能DJI设备忽然不发送视频流数据。原因很可能是没有激活应用以及绑定大疆无人机。此时你可以打开`DJI GO`做这些操作先暂时保证能正常的接收到视频流。


以下分析先暂时不考虑应用激活与绑定大疆无人机的情况，我们稍后介绍这种情况。


## 原因分析

根据解码失败的错误码，我们先后分析了多种导致这种问题的原因。

### 扔给解码器的视频数据错误

解码器返回`-12909`错误，其意义就是扔给解码器的视频数据错误。所以导致上述问题的根本原因就是视频数据错误，这是毫无疑问的。但是问题就在于我们要找出是什么原因导致了视频数据错误。

以下是视频数据的处理流程图：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171110HWDecoder/inspire_01.png)


我们首先验证了视频数据进入队列和从队列中取到的数据没有错误。具体方法是对视频数据（AVPacket中的data）进行md5加密，在进行分析data中的`NALU`之前首先验证此时获取到的视频数据的md5值是不是和进入队列之前时的md5值相同。验证结果是相同。

经过我们对代码的审查，我们发现对视频数据中的`NALU`分析处理没有发生明显的错误。

### 比较OSMO码流和inspire1码流的区别

视频数据进行`NALU`分析之前没有错误，`NALU`处理逻辑也没有问题，但是可以肯定最终给解码器的数据是有问题的，我们在代码中又没有办法检测最终传递给解码器的数据是什么。所以，此时我们怀疑`NALU`处理逻辑可能有问题。

经过分析我们发现OSMO码流的帧类型有：

Value  | type
------------- | -------------
9  | AUD
6  | SEI
1  | non-IDR
7  | sps
5  | IDR
8  | pps


inspire1码流帧类型有：

Value  | type
------------- | -------------
9  | AUD
1  | non-IDR
7  | sps
8  | pps
10  | end of sequence

而我们的处理`NALU`的逻辑是仅仅的做一个制式转化(little endian->big endian),然后部分帧类型，直接把这些数据传递给解码器了。显然这样做是有问题的，我们应该只处理帧类型为 5 和 1 的帧。

### CPU使用率过高

在进行debug调试的过程中，我们发现cpu使用率不断增加，甚至出现了在iphone 6s 上cpu占用率超过100%的情况，所以我们应该在显示DJI 设备视频画面的时候，让我们的app尽量做更少的工作。

当前的处理逻辑是：在DJI设备视频画面显示的时候，本地相机捕获视频的动作并没有停止，但是我们应该让其停止捕获。


## 最终解决方案

* 只解码帧类型为 1和5的帧。
* 在渲染DJI设备的视频流时，本地相机停止捕获。 

以下处理逻辑包含(1. 过滤帧类型 2.little endian->big endian)

```
#define NALU_TYPE_IDR  0x5
#define NALU_TYPE_SPS  0x7
#define NALU_TYPE_PPS  0x8
#define NALU_TYPE_NON_IDR 0x1
#define NALU_TYPE_AUD  0x9
#define NALU_SIZE_BYTE  4

static const uint8_t *ff_avc_find_startcode_internal(const uint8_t *p, const uint8_t *end)
{
    const uint8_t *a = p + 4 - ((intptr_t)p & 3);
    
    for (end -= 3; p < a && p < end; p++) {
        if (p[0] == 0 && p[1] == 0 && p[2] == 1)
            return p;
    }
    
    for (end -= 3; p < end; p += 4) {
        uint32_t x = *(const uint32_t*)p;
        //      if ((x - 0x01000100) & (~x) & 0x80008000) // little endian
        //      if ((x - 0x00010001) & (~x) & 0x00800080) // big endian
        if ((x - 0x01010101) & (~x) & 0x80808080) { // generic
            if (p[1] == 0) {
                if (p[0] == 0 && p[2] == 1)
                    return p;
                if (p[2] == 0 && p[3] == 1)
                    return p+1;
            }
            if (p[3] == 0) {
                if (p[2] == 0 && p[4] == 1)
                    return p+2;
                if (p[4] == 0 && p[5] == 1)
                    return p+3;
            }
        }
    }
    
    for (end += 3; p < end; p++) {
        if (p[0] == 0 && p[1] == 0 && p[2] == 1)
            return p;
    }
    
    return end + 3;
}

static const uint8_t *ff_avc_find_startcode(const uint8_t *p, const uint8_t *end){
    const uint8_t *out= ff_avc_find_startcode_internal(p, end);
    if(p<out && out<end && !out[-1]) out--;
    return out;
}



static long parseFrameIndex  = 1;
- (void)parseNalUnits:( uint8_t *)buf_in Size:(int)size validSize:(int *)len
{
    parseFrameIndex++;
    const uint8_t *p = buf_in;
    const uint8_t *end = p + size;
    const uint8_t *nal_start, *nal_end;
    
    int nal_type = 0;
    size = 0;
    nal_start = ff_avc_find_startcode(p, end);
    
//    log4cplus_info("DJIOSMOInspire1", "parse begin:------------frame index :%ld  ------------",parseFrameIndex);
    *len = 0;
    for (;;) {
        while (nal_start < end && !*(nal_start++));
        if (nal_start == end)
            break;
        
        nal_end = ff_avc_find_startcode(nal_start, end);
        uint32_t nal_size = htonl(uint32_t(nal_end - nal_start));
        //uint8_t *nal_head_convert = (uint8_t*)&nal_size;
        //printf("%02x %02x %02x %02x\n", nal_head_convert[0],nal_head_convert[1],nal_head_convert[2],nal_head_convert[3]);
        uint8_t* nal_head = (uint8_t*)(nal_start - 4);
        memcpy(nal_head, &nal_size, sizeof(uint32_t));

        nal_type = nal_start[0] & 0x1f;
        
        log4cplus_info("DJIOSMOInspire1", "parse nal, type:%d",nal_type);
        
        if (nal_type == NALU_TYPE_SPS) { /* SPS */
            _spsSize =  nal_end - nal_start;
            memcpy(_sps, nal_start, _spsSize);

            *len += (_spsSize+NALU_SIZE_BYTE);
            
        } else if (nal_type == NALU_TYPE_PPS) { /* PPS */
            _ppsSize = nal_end - nal_start;
            memcpy(_pps, nal_start, _ppsSize);

            *len += (_ppsSize+NALU_SIZE_BYTE);
            
        }else if(nal_type == NALU_TYPE_AUD){
            int aud_size = (int)(nal_end - nal_start);
            
            *len += (NALU_SIZE_BYTE+aud_size);
            
        }else if(nal_type == NALU_TYPE_IDR || nal_type ==NALU_TYPE_NON_IDR ){
  
        }
    
        size += NALU_SIZE_BYTE + nal_end - nal_start;
        nal_start = nal_end;
    }
//    log4cplus_info("DJIOSMOInspire1", "parse end:------------frame index :%ld  ------------",parseFrameIndex);
}
```

## 应用激活及大疆无人机绑定

只有在中国区域时，才需要做这步操作。具体内容可参考大疆官网 [Application Activation and Aircraft Binding](https://developer.dji.com/cn/mobile-sdk/documentation/ios-tutorials/ActivationAndBinding.html)

