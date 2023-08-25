---
layout: post
title:  查询通话状态
description: VOIP模块，查询每路电话的通话状态
category: project
tag: viop,webrtc
---

## 查询每路电话的通话状态

查询每路电话的通话状态，需要查询与`peerConnection`对应的`RTCIceConnectionState`.这个状态是一个枚举类型，有以下类型：

```
typedef NS_ENUM(NSInteger, RTCIceConnectionState) {
  RTCIceConnectionStateNew,
  RTCIceConnectionStateChecking,
  RTCIceConnectionStateConnected,
  RTCIceConnectionStateCompleted,
  RTCIceConnectionStateFailed,
  RTCIceConnectionStateDisconnected,
  RTCIceConnectionStateClosed,
  RTCIceConnectionStateCount,
};

```

当`peerConnection`对应的ice的状态变为`RTCIceConnectionStateConnected`的时候，连接建立成功，并且能听到对方声音。

参考[：RTCPeerConnection.iceConnectionState](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/iceConnectionState)



代码：

```

@property (atomic,strong) NSMutableDictionary *callStateDict;


- (KTVUVoipUserState)tvuSelectCallState:(NSString *)phoneNumber
{
    if (phoneNumber == NULL || phoneNumber == nil || [phoneNumber length] <= 0) {
        log4cplus_error("WebRTC", "the phoneNumber is null..%s",__func__);
        return KTVUVoipUserStateDisconnect;
    }
    NSString *senderID = [NSString stringWithFormat:@"ARDAMSa0%@",phoneNumber];
    KTVUVoipUserState userState = KTVUVoipUserStateDisconnect;
    @synchronized (self) {
        userState = (KTVUVoipUserState)[[_callStateDict objectForKey:senderID] intValue];
    }
    return userState;
}


/** Called any time the IceConnectionState changes. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didChangeIceConnectionState:(RTCIceConnectionState)newState
{
    NSString *senderID =[peerConnection.senders objectAtIndex:0].senderId;
    KTVUVoipUserState userState = KTVUVoipUserStateDisconnect;
    switch (newState) {
        case RTCIceConnectionStateNew:
        case RTCIceConnectionStateChecking:
            userState = KTVUVoipUserStateConnecting;
            break;
        case RTCIceConnectionStateConnected:
        case RTCIceConnectionStateCompleted:
            userState = KTVUVoipUserStateCalling;
            break;
        default:
            break;
    }
    @synchronized (self) {
        [_callStateDict setObject:[NSNumber numberWithInt:userState] forKey:senderID];
    }
}

```