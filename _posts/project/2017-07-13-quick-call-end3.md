---
layout: post
title:  问题处理-频繁打挂电话3
description: 在Anywhere中的VOIP模块，快速重复的拨打电话和挂断电话的时候，有时候 app 会发生crash
category: project
tag: viop,webrtc
---

## 现在的逻辑

### 关于`peerConnection`的创建问题

当前`peerConnection`是在收到信令服务器的`callRequest`和`callResponse`时创建的。那么这种逻辑有以下问题：

1. 如果用户点击打挂操作足够快，app在还没有收到response的时候用户就有可能又做了挂断电话的操作，所以在收到callresponse的时候创建peerconnection是不合适的，应该在用户做打电话操作的时候创建peerconnection.
2. ice状态变化时，如果用户已经挂断了电话，则剩余的步骤依然在执行。所以，应该添加一个判断，如果该路电话已被挂断，则终止处理这些代理方法之后的操作。


## 做如下改正

### 关于peerConnection的创建

1. app主动打电话时创建peerconnection，不要在app收到callresponse时才创建peerconnection.
2. 如果某路电话挂断了，则终止和该路电话相关的所有操作
3. `[peerConnection close]`的操作提前
4. 添加`[peerConnection removeMediaStream:]`,这样`senderID`会为`null`,如此省略第二步的操作。


### 挂断电话

在挂断一组电话的时候，需要把`_pcFactory=nil`,此时我们应该等所有电话都挂断的时候才做此操作。此时我们需要加一个判断，即当所有路电话都挂断完，才调用`_pcFactory=nil`,代码如下：

挂断一组电话：
```
- (void)tvuEndUpCall:(NSArray *)phoneNumbers
{
    int callNumbers = (int)phoneNumbers.count;
    if (phoneNumbers == NULL || phoneNumbers == nil || [phoneNumbers count] <= 0) {
        log4cplus_error("WebRTCLog", "The end up call request array is null..%s",__func__);
        self.endCallNumbers = 0;
        return;
    }
    self.isEndupCall = YES;
    
    _tvuSignal->ClearMessageQueue();
    
    for (NSString *phone in phoneNumbers) {
        
        dispatch_queue_t phoneQueue = [self getSerialQueue:phone];
        dispatch_async(phoneQueue, ^{
            if (phone == self.nowCallFromPhone) {
                self.nowCallFromPhone = NULL;
            }
            [self hangupPhone:phone];
        });
    }
    
    dispatch_queue_t phoneQueue = [self getSerialQueue:(NSString *)[phoneNumbers lastObject]];
    dispatch_async(phoneQueue, ^{
        
        while (self.endCallNumbers < callNumbers) {
            usleep(1000*10);
            log4cplus_warn("WebRTCLog", "%s,line=%d,endCallNumbers=%d,callNumbers =%d,endCallNumbers < callNumbers,continue",__func__,__LINE__,self.endCallNumbers,callNumbers);
        }
        
        _pcFactory = nil;
        usleep(1000*500);
        
        log4cplus_warn("WebRTCLog", "%s,line=%d,endCallNumbers=%d callNumbers=%d,_pcFactory=nil...",__func__,__LINE__,self.endCallNumbers,callNumbers);
        self.endCallNumbers = 0;
    });
    
    dispatch_async(TVUMainQueue, ^{
        [self.delegate endupCallWithTVUWebRTCManager:self];
    });
}

```

挂断单路电话：
```
- (void)hangupPhone:(NSString *)phone
{
    log4cplus_warn("WebRTCLog", "%s,hang up phone,phone=%s",__func__,[phone UTF8String]);
    if (phone != NULL) {
        _tvuSignal->postDisconnectpeer([phone UTF8String]);
    }
    __block RTCPeerConnection *peerConnection = [self getPeerConnectionUsePhoneNumber:phone];
    if (!peerConnection) {
        log4cplus_error("WebRTCLog", "hang up phone error,phone=%s",[phone UTF8String]);
        self.endCallNumbers++;
        return;
    }
    NSArray *streamsArray = peerConnection.localStreams;
    
    if (streamsArray || [streamsArray count] > 0) {
        RTCMediaStream *mediaStream = [peerConnection.localStreams firstObject];
        if (mediaStream) {
            log4cplus_error("WebRTCLog", "%s,remove mediaStream,phone=%s,line=%d",__func__,[phone UTF8String],__LINE__);
            [peerConnection removeStream:mediaStream];
        }
    }
    
    [peerConnection close];
    [self removeElementsFromPeerConnectionsDict:phone andType:KRemovePeerConnectionTypeEndupCall];
    usleep(1000*100);
    
    peerConnection = nil;
    log4cplus_warn("WebRTCLog", "%s,peerConn=nil,phone=%s,line=%d",__func__,[phone UTF8String],__LINE__);
    self.endCallNumbers++;
}
```

