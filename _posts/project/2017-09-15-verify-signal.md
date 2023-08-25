---
layout: post
title:  问题处理-过滤信令
description: VOIP模块，在Anywhere的TVUCC界面和其它设备(TVUPACK,Anywhere)进行viop电话。没有成功挂断并且一旦用户离开主页面，在app端就能听到其它T的声音
category: project
tag: viop,webrtc
---
## bug场景

在`Anywhere`的TVUCC界面和其它设备(TVUPACK,Anywhere)进行viop电话。具体操作是：

* 选择一组拨打
* 还没等打通立马挂断这组电话

现象：没有成功挂断。并且一旦用户离开主页面，在app端就能听到其它T的声音。经过检测，发现app一直保持通话中。

## 分析原因

通过查看信令服务器的log我们发现：

* Anywhere拨打了一组电话(4路)，首先发送call_request .
* Anywhere还没有等到另一端的call_response，就已经挂断了电话即发送了disconnect指令。
* 另一端(TVUPACK)忽略了Anywhere此时发送的disconnect信令
* 另一端(TVUPack)此时发送了call_response，`Anywhere`端接收到该信令后开始创建peerConnection.

`Anywhere`端打电话的逻辑是当收到另一端的response时，创建peerConnection，然后做其余的和打电话相关的操作。

## 原因归属

发生以上bug的原因归属于三个方面，其中任何一方解决问题都可以避免该bug。具体有：
* 信令服务器：当a拨打b进行voip通话时，无论有没有打通，只要a发送了disconnect信令，那么当b再给a发送除call_request指令以外的任何指令都应该被禁止，因为a已经disconnect。（但是，我们的信令服务器）

* 其它的T(TVUPACK): 打电话和挂电话的操作间隔可以非常接近，用户是可以频繁的打挂电话的。用户挂电话的时候并不一定是该电话通的时候，此种case就是用户打电话时突然间又取消了打电话的操作。所以它在这种case下不应该忽略Anywhere端发送的disconnect信令。

* Anywhere端：在手机端发送了打电话指令`call_request`，还没有等到对方回`call_response`时就给对方发送了`disconnect`指令。当`disconnect`指令发送了之后，相当于本次打电话逻辑的终结，即这个事物已经完成。当此时再接收到诸如`call_response`、`offer`、`ice`、`answer`等信令时，应该忽略。


此时我们只负责处理Anywhere端的问题。处理这种问题其实就是有效信令的过滤。当我们接收到一条信令时，我们是不是应该忽略它。

## 解决问题

### case1:为每一路电话添加状态标记

这是一种非常简单的过滤方法。具体逻辑是：

* 无论是打电话还是接电话，为每一路电话都做一个状态标记。即：打\接的状态，还是已经挂断的状态。
* 在打电话操作中，设置该路电话为打|接状态(call)
* 在接听电话的操作中，设置该路电话为打|接状态(call)
* 在用户主动挂断电话时，设置该路电话为已经挂断状态(disconnect)
* 在用户被动挂断(即用户接收到disconnect信令时)，设置该路电话为已经挂断状态(disconnect)


* 在监听信令消息时，如果信令类型为`offer,ice,answer,response`，则需要判断该路电话当前的状态是不是call,如果是call,则处理该信令，如果不是call,则忽略该信令。


## 代码实现：

1） 用一个map保存每一路电话的当前状态。key是电话号码，value是状态(`KTVUVoiceState`类型)

每一路电话状态`KTVUVoiceState`的枚举定义：

```
typedef enum
{
    KTVUVoiceState_unknown = 0,
    KTVUVoiceState_call,
    KTVUVoiceState_disconnect
}KTVUVoiceState;
```

2) 对map的操作：

```
#pragma mark --control signaling state of call
- (void)updateSignalingStateWithPhone:(NSString *)phone andState:(KTVUVoiceState)state
{
    log4cplus_warn("WebRTCLog", "%s change state to %d",[phone UTF8String],(int)state);
    [self.signalStateDict setObject:[NSNumber numberWithInt:(int)state] forKey:phone];
}

- (KTVUVoiceState)getSignalingStateOfPhone:(NSString *)phone
{
    NSNumber *state = [self.signalStateDict objectForKey:phone];
    if (state == NULL) {
        return KTVUVoiceState_unknown;
    }
    return (KTVUVoiceState)[state intValue];
}

- (BOOL)shouldDismissThisSignal:(NSString *)phone
{
    KTVUVoiceState state = [self getSignalingStateOfPhone:phone];
    if (state == KTVUVoiceState_disconnect) {
        log4cplus_warn("WebRTCLog", "%s,%s state is KTVUVoiceState_disconnect,dismiss this signal..",__func__,[phone UTF8String]);
        return YES;
    }
    return NO;
}

```

3) 信令过滤

```
if (messtype == KSignalingTypeOffer || messtype == KSignalingTypeIce || messtype == KSignalingTypeCallResponse || KSignalingTypeAnswer) {
        
        if ([self shouldDismissThisSignal:callFromNumber]) {
            return;
        }
        
    }

```

4) 更新每路电话状态：

设置为call状态（收到call_request时）
```
case KSignalingTypeCallRequest:
        {
            log4cplus_warn("WebRTCLog", "%s,receive call request info, message=%s",__func__,[callFromNumber UTF8String]);
            
            dispatch_queue_t phoneQueue = [self getSerialQueue:callFromNumber];
            dispatch_async(phoneQueue, ^{
                [self updateSignalingStateWithPhone:callFromNumber andState:KTVUVoiceState_call];   // set phone state is call
            });
            
            [self processCallRequestUseMessageData:messageStr andPhoneNumber:callFromNumber];
        }
            break;
```

设置为call状态（主动打电话时）
```
- (void)callPhone:(NSString *)phone
{
    log4cplus_warn("WebRTCLog", "%s,call request,data:%s",__func__,[phone UTF8String]);

    NSString *phones = [NSJSONSerialization JSONStringWithJSONObject:@[phone]];
    if (!phones) {
        log4cplus_error("WebRTCLog", "%s,call phone error,phone is null",__func__);
        return;
    }
    _tvuSignal->postCallRequest([phones UTF8String]);
    
    [self updateSignalingStateWithPhone:phone andState:KTVUVoiceState_call];
}
```

设置为disconnect状态（被动挂电话时--收到disconnect信令）
```
- (void)processDisconnectPeerUseMessageData:(NSString *)message andPhoneNumber:(NSString *)phoneNumber
{
    self.isEndupCall = YES;
    
    dispatch_queue_t phoneQueue = [self getSerialQueue:phoneNumber];
    dispatch_async(phoneQueue, ^{
        __block RTCPeerConnection *peerConnection = [self getPeerConnectionUsePhoneNumber:phoneNumber];
        
        if (!peerConnection) {
            log4cplus_warn("WebRTCLog", "%s,cancel call...",__func__);
            [self.delegate cancelCallWithTVUWebRTCManager:self];
            return;
        }
        
        if (phoneNumber == self.nowCallFromPhone) {
            self.nowCallFromPhone = NULL;
        }

        usleep(1000*100);
        [self removeElementsFromPeerConnectionsDict:phoneNumber andType:KRemovePeerConnectionTypeDisconnect];
        [peerConnection close];
        [peerConnection stopRtcEventLog];
        peerConnection = nil;
        log4cplus_warn("WebRTCLog", "%s,peerConn close,_peerConn=nil, phone=%s",__func__,[phoneNumber UTF8String]);
        [self.delegate endupCallWithTVUWebRTCManager:self];
    });

}
```

设置为disconnect状态（主动挂断电话时）
```
- (void)hangupPhone:(NSString *)phone
{
    log4cplus_warn("WebRTCLog", "%s,hang up phone,phone=%s",__func__,[phone UTF8String]);
    if (phone != NULL) {
        _tvuSignal->postDisconnectpeer([phone UTF8String]);
        [self updateSignalingStateWithPhone:phone andState:KTVUVoiceState_disconnect];
    }
    __block RTCPeerConnection *peerConnection = [self getPeerConnectionUsePhoneNumber:phone];
    if (!peerConnection) {
        log4cplus_error("WebRTCLog", "hang up phone error,phone=%s",[phone UTF8String]);
        return;
    }
    NSArray *streamsArray = peerConnection.localStreams;
    
    if (streamsArray || [streamsArray count] > 0) {
        RTCMediaStream *mediaStream = [peerConnection.localStreams firstObject];
        if (mediaStream) {
            log4cplus_error("WebRTCLog", "%s,remove mediaStream,phone=%s,line=%d",__func__,[phone UTF8String],__LINE__);
//            [peerConnection removeStream:mediaStream];
        }
    }
    
    [peerConnection close];
    [self removeElementsFromPeerConnectionsDict:phone andType:KRemovePeerConnectionTypeEndupCall];
    usleep(1000*100);
    
    [peerConnection stopRtcEventLog];
    peerConnection = nil;
    log4cplus_warn("WebRTCLog", "%s,peerConn=nil,phone=%s,line=%d",__func__,[phone UTF8String],__LINE__);
}

```

