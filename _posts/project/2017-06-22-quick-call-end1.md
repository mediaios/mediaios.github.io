---
layout: post
title:  问题处理-频繁打挂电话
description: 在Anywhere中的VOIP模块，快速重复的拨打电话和挂断电话的时候，有时候 app 会发生crash
category: project
tag: viop,webrtc
---

## 场景描述

### 问题的出现

在`Anywhere`中的`VOIP`模块，快速重复的拨打电话和挂断电话的时候，有时候 app 会发生crash

经过分析我们发现，挂断电话和拨打电话时都对`peerConnection`进行操作，重复快速的做拨挂电话会导致同一个电话号码的`peerConnection`资源不同步的问题。

### 对现有处理逻辑的说明

**拨打电话：arry存储的是一些列电话号码-phone**
callRequest:array |
---|---
112 | 
113 | 
114 |

处理逻辑：

如果phone对应的`peerConn`已经存在，则认为其正在通话中，不再进行拨打电话动作；如果不存在，则在收到对方的response后创建`peerConn`


**挂断电话:arry存储的是一些列电话号码-phone**

endupCall:array |
---|---
112 |
113 |
114 |

处理逻辑：

* 清空信令消息队列
* 如果phone对应的`peerConn`为空，说明已经挂断了电话，不再进行挂断电话的动作；如果存在，则从`peerConn池子`中删除该`peerConn`
* 删除`peerConn`之后，让`peerConn`销毁，即`peerConn=nil`
* crash就发生在了这里,在`peerConn=nil时`。

### 对crash的分析

在挂断电话操作中，当`peerConn=nil`时，发生了crash。这说明当挂断电话时，该`peerConn`还没有被移除时又拨打了该电话，正在做拨打电话的动作时(会用到peerConn),`peerConn=nil`了，这就导致了app的crash .


## 解决问题

解决问题的方法有很多，但本质上无非从资源同步和操作控制两个方面入手。

### 方法一(资源同步)

可以让每一个phone对应的`peerConn`保持资源同步。

### 方法二(利用消息队列)


利用消息队列是避免资源不同步的一个方法，我们可以把所有的指令操作都放到一个消息队列中，然后在一个线程中去检测消息对了中的指令然后䦹相应的处理。

这种方法我已经用代码实现了，但是最终觉得用这种方式不太合适，原因如下：
* 每次处理只能处理一个电话，对于处理一组电话的余后操作(比如：pcFactory=nil,设置AudioPlayer的播放模式等)不太方便处理，只有通过检测消息队列的内容是否为空，做这些处理。
* 这种方法需要启动一个检测消息队列的线程，即app一启动就要开启这个线程来检测消息队列的内容，用户可以在没有登录webrtc服务器就做打挂电话的操作。
* 如果app是被动接听电话，那么还要把被动接听的动作添加倒消息队列中。
* 多启动了一个检测线程，这本质上就是一种开销。并且还不能避免`peerConn及peerConn池`的不同步。

### 方法三(为每一路电话添加状态信息)

对于每一路电话都添加一个状态属性，这个状态属性是一个枚举值：

```
typedef enum
{
    KTVUVoipCommandType_dialing = 0,  // dialing
    KTVUVoipCommandType_ending,       // ending
    KTVUVoipCommandType_idle     // normal state,can accept instruction
}KTVUVoipCommandType;

```

以上状态转换规则:



* 拨打电话：拨打开始时，设置状态为dailing,拨打完成，设置状态为idle
* 挂断电话: 挂断电话开始是，设置状态为ending，挂断完成时，设置状态为idle

具体处理逻辑：

1. 在做拨打电话和挂断电话操作前，都要判断一下对应`phone`的状态是否为`accepting`，如果不是`accepting`，则再进行三次判断，每次判断的时间间隔为1秒。具体伪代码为：
    
    打电话：
    ```
    call function:phone
    if(phone->state==accepting){
        endup/call;
    }else if(phone->state==ending){
        while(true && times<=3)
        {
            if(phone->state==accepting){
                endup/call;
                continue;
            }else{
                sleep(1second);
                times++;
            }
            if(times > 3){
                break;
            }
        }
    }else{
        return;
    }
    ```
    挂电话：
    ```
    endup function:phone
    if(phone->state==accepting){
        endup/call;
    }else if(phone->state==dailing){
        while(true && times<=3)
        {
            if(phone->state==accepting){
                endup/call;
                continue;
            }else{
                sleep(1second);
                times++;
            }
            if(times > 3){
                break;
            }
        }
    }else{
        return;
    }
    
    ```
  打挂电话做完之后，设置phone状态为accepting .

2. phone->state的存储

   用一个字典`phoneStateDict`存储每一个拨打电话的状态。`phoneStateDict`随着app的启动创建，随着app的退出而销毁。这个地方也需要资源同步。

       key  |    value  |
    ---|---
    phone | state |
    
3. 由于问题出现的原因是频繁的拨挂电话所造成的，因此我们应该从源头上尽量避免问题的出现。可以在UI上做控制——即拨挂电话的时间间隔至少是3秒（按钮点击一下3s后才可用）,因为拨打电话和挂断电话是一个事件，处理事件总需要写时间处理。
4. 为什么不把拨挂电话的时间间隔放在底层去处理？如果在底层去处理拨打电话的时间间隔的话是很不科学的。因为如果不控制上层的按钮交互，那么1s中有可能点击拨打电话挂断按钮很多次(例如：一些压力测试软件或者一些脚本可以1秒钟点击一个按钮上千次),那么我们底层就会积累很多待处理的拨挂事件，如果这些事件的处理都有一个事件间隔，那么可能需要很久才能处理完成，但是在这段处理的时间内用户有可能在做其它的操作，所以这样是很不科学的。如果同一个号码在3s内多次做拨挂电话，纵然你可以让这些事件取消，但是仍然不合理，因为按钮接受事件本身就是一个损耗性能的多余处理。


#### 具体代码：

```
@property (atomic,strong) NSMutableDictionary *commandStateDict; // store endup/call state


typedef enum
{
    KTVUVoipCommandType_dialing = 0,  // dialing
    KTVUVoipCommandType_ending,       // ending
    KTVUVoipCommandType_idle     // normal state,can accept instruction
}KTVUVoipCommandType;

- (NSMutableArray *)storeArrayphoneState:(NSArray *)phoneNumbers andState:(KTVUVoipCommandType)state
{
    NSMutableArray *phonesArray = [NSMutableArray arrayWithArray:phoneNumbers];
    @synchronized (self) {
        switch (state) {
            case KTVUVoipCommandType_dialing:
            {
                
                for (NSString *phoneNumber in phoneNumbers) {
                    NSNumber *number = [_commandStateDict objectForKey:phoneNumber];
                    if (!number) { // phone is not exist
                        [_commandStateDict setObject:[NSNumber numberWithInteger:KTVUVoipCommandType_dialing] forKey:phoneNumber];
                    }else{
                        
                        KTVUVoipCommandType phoneState = (KTVUVoipCommandType)number.intValue;
                        if (phoneState == KTVUVoipCommandType_idle) {
                            [_commandStateDict setObject:[NSNumber numberWithInteger:KTVUVoipCommandType_dialing] forKey:phoneNumber];
                        }else if (phoneState == KTVUVoipCommandType_ending){
                            [self handleUnnormalPhoneCall:phoneNumber commandType:KTVUVoipCommandType_dialing phoneArray:phonesArray];
                        }else{
                            log4cplus_error("WebRTC", "%s, the %s state is calling,remove this call...",__func__,[phoneNumber UTF8String]);
                            [phonesArray removeObject:phoneNumber];
                        }
                    }
                }
                
//                NSLog(@"qizhang---debug---dailing---%@",_phoneStateDict);
            }
                break;
            case KTVUVoipCommandType_ending:
            {
                for (NSString *phoneNumber in phoneNumbers) {
                    NSNumber *number = [_commandStateDict objectForKey:phoneNumber];
                    
                    KTVUVoipCommandType phoneState = (KTVUVoipCommandType)number.intValue;
                    if (phoneState == KTVUVoipCommandType_idle) {
                        [_commandStateDict setObject:[NSNumber numberWithInteger:KTVUVoipCommandType_ending] forKey:phoneNumber];
                    }else if (phoneState == KTVUVoipCommandType_dialing){
                        [self handleUnnormalPhoneCall:phoneNumber commandType:KTVUVoipCommandType_ending phoneArray:phonesArray];
                    }else{
                        log4cplus_error("WebRTC", "%s, the %s state is unnormal,remove this call...",__func__,[phoneNumber UTF8String]);
                        [phonesArray removeObject:phoneNumber];
                    }
                }
//                NSLog(@"qizhang---debug---ending---%@",_phoneStateDict);
            }
                break;
            case KTVUVoipCommandType_idle:
            {
                for (NSString *phoneNumber in phoneNumbers) {
                    [_commandStateDict setObject:[NSNumber numberWithInteger:KTVUVoipCommandType_idle] forKey:phoneNumber];
                }
            }
                break;
                
            default:
                break;
        }
    }
    return phonesArray;
}



#pragma mark -call function
- (void)tvuCallRequest:(NSArray *)phoneNumbers
{
    if (phoneNumbers == NULL || phoneNumbers == nil || [phoneNumbers count] <= 0) {
        log4cplus_error("WebRTC", "The call reqeust array is null..%s",__func__);
        return;
    }
    
    phoneNumbers = [self storeArrayphoneState:phoneNumbers andState:KTVUVoipCommandType_dialing];
    
    dispatch_async(TVUGlobalQueue, ^{
        NSMutableArray *needCallPhones = [NSMutableArray array];
        for (NSString *phone in phoneNumbers) {
            RTCPeerConnection *peerConnection = [self getPeerConnectionUsePhoneNumber:phone];
            TVULOCK(_tvuWebRTCLock);
            if (peerConnection != NULL) {
                TVUUNLOCK(_tvuWebRTCLock);
                continue;
            }else{
                TVUUNLOCK(_tvuWebRTCLock);
                [needCallPhones addObject:phone];
            }
        }
        
        if (needCallPhones.count == 0) {
            return;
        }
        
        NSString *phones = [NSJSONSerialization JSONStringWithJSONObject:needCallPhones];
        _tvuSignal->postCallRequest([phones UTF8String]);
        
        
        [self storeArrayphoneState:phoneNumbers andState:KTVUVoipCommandType_idle];
        
    });
}

- (void)tvuEndUpCall:(NSArray *)phoneNumbers
{
    if (phoneNumbers == NULL || phoneNumbers == nil || [phoneNumbers count] <= 0) {
        log4cplus_error("WebRTC", "The end up call request array is null..%s",__func__);
        return;
    }
    self.isEndupCall = YES;
    
    phoneNumbers = [self storeArrayphoneState:phoneNumbers andState:KTVUVoipCommandType_dialing];
    
    dispatch_async(TVUGlobalQueue, ^{
        _tvuSignal->ClearMessageQueue();
        
        for (NSString *phone in phoneNumbers) {
           __block RTCPeerConnection *peerConnection = [self getPeerConnectionUsePhoneNumber:phone];
            TVULOCK(_tvuWebRTCLock);
            if (peerConnection == NULL) {
                TVUUNLOCK(_tvuWebRTCLock);
                continue;
            }
            TVUUNLOCK(_tvuWebRTCLock);
        
            if (phone == self.nowCallFromPhone) {
                self.nowCallFromPhone = NULL;
                dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, 100*NSEC_PER_MSEC);
                dispatch_after(time, TVUGlobalQueue, ^{
                    [self removeElementsFromPeerConnectionsDict:phone andType:KRemovePeerConnectionTypeEndupCall];
                    _tvuSignal->postDisconnectpeer([phone UTF8String]);
                    [peerConnection close];
                    peerConnection = nil;
                });
            }else{
                [self removeElementsFromPeerConnectionsDict:phone andType:KRemovePeerConnectionTypeEndupCall];
                _tvuSignal->postDisconnectpeer([phone UTF8String]);
                [peerConnection close];
                peerConnection = nil;
            }
        }
        _pcFactory = nil;
        
        [self storeArrayphoneState:phoneNumbers andState:KTVUVoipCommandType_idle];
        
        dispatch_async(TVUMainQueue, ^{
            [self.delegate endupCallWithTVUWebRTCManager:self];
        });
    });
}


- (void)handleUnnormalPhoneCall:(NSString *)phone commandType:(KTVUVoipCommandType)type phoneArray:(NSMutableArray *)phoneArray
{
    int times = 1;
    while (times <= 3) {
            NSNumber *number = [_commandStateDict objectForKey:phone];
            KTVUVoipCommandType phoneState = (KTVUVoipCommandType)number.intValue;
            if (phoneState == KTVUVoipCommandType_idle) {
                [_commandStateDict setObject:[NSNumber numberWithInteger:type] forKey:phone];
                break;
            }else{
                times++;
                usleep(1000*1000); // sleep 1 second
            }
            if (times > 3) {
                log4cplus_error("WebRTC", "%s, the %s state is unnormal,remove this call...",__func__,[phone UTF8String]);
                [phoneArray removeObject:phone];
                break;
            }
    }
}



```