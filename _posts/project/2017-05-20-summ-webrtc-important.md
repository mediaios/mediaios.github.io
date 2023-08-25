---
layout: post
title: WebRTC视频通话技术关键点
description: 总结WebRTC视频通话技术实现
category: project
tag: ios, webrtc, Swift,voip
---

## 说明

这篇文章主要介绍Anywhere RPS系统的需求、功能实现设计等内容。里面附带有`WebRTC`视频通话技术关键点。下面是详细内容。

### Anywhere RPS需求

Anywhere的新需求：需要做一个远端产品系统—RPS(Remote Product System)。用于在手机端展示不同T端的视频，类似于一个网格监控系统，一个网格代表一路音视频通话。具体可描述为：

* 把手机端(T端)的视频发送给服务端，即服务端能显示T端的视频，并且能听到声音
* 手机端显示每一路通话的视频，用一个Grid来展示各路视频

### 使用到的技术及集成思路

具体技术就是集成`WebRTC`的音视频。RPS系统是一个多路监控系统。具体的功能可描述为：(在RPS模块)

* 手机端展示各路的通话画面，可以根据通话路数自动计算远端视频画面在手机端如何排版
* 手机端是否要展示本地采集的视频画面
* 当手机当前页面处在RPS页面时，是自动接听电话，还是主动发出打电话请求？如果是被动电话，是否可以拒绝？

## 集成WebRTC视频通话

相较于集成`WebRTC`的语音通话，集成其视频通话同样也需要集成其语音通话时的一些必要流程，其具体内容可参考以前总结的语音通话的内容。

下面我罗列出集成`WebRTC`视频通话的关键点。

### 传输手机端视频

如果在R端想要接受手机端的视频，那么必须让手机端去捕获视频传送给R端。那么在创建`peerConnection`的时候，需要为`peerConnection`添加媒体流。这些媒体流就是用于采集音视频信息。

`RTCMediaStream`是媒体流，当集成视频通话时，需要为`localStream`(本地媒体流)添加视频信息。

具体代码：

```
/*
如果videoSource在创建mediaStream的方法中创建（videoSource是一个局部变量时），那么只能成功创建一路音视频通话，其它路会断开。所以在进行多路通话时，一定要把videoSource当做一个全局变量，用其去创建videoTrack
*/
- (RTCVideoSource *)videoSource
{
    if (!_videoSource) {
        RTCMediaConstraints *mediaConstraints = [[RTCMediaConstraints alloc] initWithMandatoryConstraints:nil optionalConstraints:nil];
        _videoSource = [self.pcFactory avFoundationVideoSourceWithConstraints:mediaConstraints];
    }
    return _videoSource;
}
    
- (RTCMediaStream *)createStreamUsePhoneNumber:(NSString *)phoneNumber
{
    log4cplus_debug("WebRTCLog", "create media stream , phone=%s",[phoneNumber UTF8String]);
    RTCMediaStream *mediaStream = [self.pcFactory mediaStreamWithStreamId:[NSString stringWithFormat:@"ARDAMS%@",phoneNumber]];;
    RTCAudioTrack *audioTrack = [self.pcFactory audioTrackWithTrackId:[NSString stringWithFormat:@"%@%@",KAudioTrackIDPrefix,phoneNumber]];
    [mediaStream addAudioTrack:audioTrack];
  
    /* support video  */
    RTCVideoTrack *videoTrack = [self.pcFactory videoTrackWithSource:self.videoSource trackId:[NSString stringWithFormat:@"ARDAMSv0%@",phoneNumber]];
    [mediaStream addVideoTrack:videoTrack];
    return mediaStream;
}

```

### WebRTC视频的显示

在手机端，可以显示手机自己采集的视频，同时也能显示R端的视频。`WebRTC`为我们提供好了视频渲染的`View`类，我们只需要创建`localView`和`remoteView`，然后把这些`view`交给`WebRTC`让它去渲染就可以了。

`WebRTC`中用于视频渲染的类和方法：

* 显示视频的view类是`RTCEAGLVideoView`，该类有一个代理是`RTCEAGLVideoViewDelegate`
* 我们可以分别定义localView和remoteView，都是`RTCEAGLVideoView`类型的，用于显示本地视频和R端视频
* 把`RTCEAGLVideoView`实例传给`RTCVideoTrack`的`- (void)addRenderer:(id<RTCVideoRenderer>)renderer`方法来注册一个渲染器
* 用localMediaStream的`RTCVideoTrack`实例注册本地视频的渲染器，用remoteMedaStream的`RTCVideoTrack`实例注册R端视频的渲染器。

```
//
//  TVUWebRTCManager.h
//  TVUAnywhere
//
//  Created by zhangqi on 5/1/2017.
//
//

#import <Foundation/Foundation.h>
#import "TVUConst.h"
#import "TVUSignaling.h"
#import <WebRTC/WebRTC.h>
#import "NSJSONSerialization+TVU.h"
#import "RTCIceCandidate+JSON.h"
#import "TVUWebRTCPeerConnectionDescription.h"

#import "VoipAVViewController.h"

@class TVUWebRTCManager;

@protocol TVUWebRTCManagerDelegate <NSObject>

@optional
- (void)acceptCallWithTVUWebRTCManager:(TVUWebRTCManager *)tvuWebRTCManager PhoneNumber:(NSString *)phoneNumber;
- (void)endupCallWithTVUWebRTCManager:(TVUWebRTCManager *)tvuWebRTCManager;
- (void)cancelCallWithTVUWebRTCManager:(TVUWebRTCManager *)tvuWebRTCManager;

- (void)receiveRemoteVideoStream:(RTCVideoTrack *)videoTrack phoneNumber:(NSString *)phone;
- (void)localVideoTrack:(RTCVideoTrack *)videoTrack phoneNumber:(NSString *)phone;
@end

@interface TVUWebRTCManager : NSObject

@property (nonatomic,strong) id<TVUWebRTCManagerDelegate> delegate;

/* Marks whether the call ends and is used for resource synchronization */
@property (nonatomic,assign) BOOL isEndupCall;

+ (instancetype)shareInstance;
- (void)tvuSettingWebRTCSDKType:(KTypeWebRTCSDK)type;
- (void)loginWebRTCServer:(NSString *)peerid;
- (RTCPeerConnection *)getNewestPeerConnection;
- (void)beginCommiuncationWithPhoneNumber:(NSString *)phoneNumber isAccept:(BOOL)accept;

- (void)tvuCallRequest:(NSArray *)phoneNumbers;
- (void)tvuEndUpCall:(NSArray *)phoneNumbers;
- (KTVUVoipUserState)tvuSelectCallState:(NSString *)phoneNumber;
- (BOOL)voipIsCalling;
- (void)tvuWriteAudiodataToWebRTC:(int8_t *)data dataSize:(size_t)size writeTime:(float)writeTime;
- (BOOL)isCallEndupMethod;
- (void)tvuAcceptCall:(NSString *)phone;
- (void)tvuRejectCall:(NSString *)phone;
- (void)tvuHangupCall:(NSString *)phone;
- (void)settingPcFactoryToNil;
- (int)getWaitingTime;

@property (nonatomic,strong) RTCVideoTrack *localVideoTrack;



// test
//- (void)testCallAndEndupFunc;
//- (void)testHangupcall;
@end


```

```
//
//  TVUWebRTCManager.m
//  TVUAnywhere
//
//  Created by zhangqi on 5/1/2017.
//
//

#import "TVUWebRTCManager.h"
#include "log4cplus.h"
#import "TVUDateTool.h"
//#import "NSLock+TVU.h"

#define kStunserver1 @"turn:rtcturn.tvunetworks.com:13478"
#define kStunserver2 @"turn:rtc.tvunetworks.com:13479"
#define kStunserver3 @"stun:stun.l.google.com:19302"
#define  kStunserver4 @"stun:stun.sipgate.net"

#define KAudioTrackIDPrefix @"ARDAMSa0"
#define kSerialQueueTag @"TVUWebRTCQueue"
#define kSwitchWebRTCSDKTime 5

#define TVULOCK(weblock)\
do{\
    [weblock lock];\
    log4cplus_warn("rtclock","%s----lock-----%d",__func__,__LINE__);\
}while(0)

#define TVUUNLOCK(weblock)\
do{\
    [weblock unlock];\
    log4cplus_warn("rtclock","%s----unlock-----%d",__func__,__LINE__);\
}while(0)

@interface TVUWebRTCManager() <RTCPeerConnectionDelegate>
{
    TVUSignaling *_tvuSignal;
}

@property (nonatomic,strong) RTCPeerConnectionFactory *pcFactory;
@property (nonatomic,strong) NSString *nowCallFromPhone;
@property (nonatomic,strong) NSLock *tvuWebRTCLock;
@property (nonatomic,strong) NSMutableDictionary *peerConnectionsDict;
@property (atomic,strong) NSMutableDictionary *callStateDict;
@property (nonatomic,strong) NSMutableDictionary *serialQueueDict;
@property (nonatomic,strong) NSMutableDictionary *signalStateDict; /* save the current signaling states of each call */

@property (nonatomic,assign) BOOL isCallEndupMethod;  /* judge  */
@property (nonatomic,assign) double pcFactoryIsNilTime;
@property (nonatomic,strong) NSTimer *signalMessageTimer;

@property (nonatomic,strong) RTCAVFoundationVideoSource *videoSource;
@end

@implementation TVUWebRTCManager

static TVUWebRTCManager *tvuWebRTCManager_instance = NULL;

#pragma mark --system method
- (RTCPeerConnectionFactory *)pcFactory
{
    if (!_pcFactory) {
        _pcFactory = [[RTCPeerConnectionFactory alloc] init];
        RTCSetMinDebugLogLevel(RTCLoggingSeverityVerbose);
//        RTCSetMinDebugLogLevel(RTCLoggingSeverityError);
    }
    return _pcFactory;
}

- (NSMutableDictionary *)serialQueueDict
{
    if (!_serialQueueDict) {
        _serialQueueDict = [NSMutableDictionary dictionary];
    }
    return _serialQueueDict;
}

- (NSMutableDictionary *)signalStateDict
{
    if (!_signalStateDict) {
        _signalStateDict = [NSMutableDictionary dictionary];
    }
    return _signalStateDict;
}

- (RTCVideoSource *)videoSource
{
    if (!_videoSource) {
        RTCMediaConstraints *mediaConstraints = [[RTCMediaConstraints alloc] initWithMandatoryConstraints:nil optionalConstraints:nil];
        _videoSource = [self.pcFactory avFoundationVideoSourceWithConstraints:mediaConstraints];
    }
    return _videoSource;
}

- (instancetype)init
{
    if (self = [super init]) {
        _tvuSignal = new TVUSignaling();
        _peerConnectionsDict = [NSMutableDictionary dictionary];
        _tvuWebRTCLock = [[NSLock alloc] init];
        _callStateDict = [NSMutableDictionary dictionary];
        _signalMessageTimer = [NSTimer scheduledTimerWithTimeInterval:0.2 target:self selector:@selector(processingMessageQueue) userInfo:nil repeats:YES];
    }
    return self;
}

+ (instancetype)shareInstance
{
    if (tvuWebRTCManager_instance == NULL) {
        tvuWebRTCManager_instance = [[TVUWebRTCManager alloc] init];
    }
    return tvuWebRTCManager_instance;
}

- (void)stopSignalMessageTimer
{
    [_signalMessageTimer setFireDate:[NSDate distantFuture]];
    log4cplus_warn("WebRTCLog", "%s",__func__);
}

- (void)startSignalMessageTimer
{
    [_signalMessageTimer setFireDate:[NSDate date]];
    log4cplus_warn("WebRTCLog", "%s",__func__);
}

- (dispatch_queue_t)getSerialQueue:(NSString *)phone
{
    phone = @"tvuvoipQueue";
    dispatch_queue_t result = NULL;
    if (phone == NULL || [phone length] <= 0) {
        return result;
    }
    
    result = [self.serialQueueDict objectForKey:phone];
    if (result == NULL) {
        NSString *queueID = [NSString stringWithFormat:@"%@-serialQueue",phone];
        result = dispatch_queue_create([queueID UTF8String], DISPATCH_QUEUE_SERIAL);
        [self.serialQueueDict setObject:result forKey:phone];
        log4cplus_debug("WebRTCLog", "create serial queue,name=%s",[queueID UTF8String]);
    }
    return result;
}

- (void)loginWebRTCServer:(NSString *)peerid
{
    if (peerid == NULL || peerid == nil || [peerid length] <= 0) {
        log4cplus_error("WebRTCLog", "The parameter of peerid is null..%s",__func__);
        return;
    }
    log4cplus_warn("WebRTCLog", "login WebRTC server,peerid=%s",[peerid UTF8String]);
    _tvuSignal->setTvuusernumber(std::string([peerid UTF8String]));
    dispatch_async(TVUGlobalQueue, ^{
        _tvuSignal->beginConnection();
    });
}

#pragma mark -public interface
- (void)tvuCallRequest:(NSArray *)phoneNumbers
{
    if (phoneNumbers == NULL || phoneNumbers == nil || [phoneNumbers count] <= 0) {
        log4cplus_error("WebRTCLog", "The call reqeust array is null..%s",__func__);
        return;
    }
    
    dispatch_queue_t queue = [self getSerialQueue:kSerialQueueTag];
    dispatch_async(queue, ^{
        [self checkSwitchWebRTCSDKTimestamp];
        [self tvuSettingWebRTCSDKType:KTypeWebRTCSDK_Original];
        
        for (NSString *phone in phoneNumbers) {
            [self callPhone:phone];
        }
        
    });
    
}

- (void)tvuHangupCall:(NSString *)phone
{
    dispatch_queue_t queue = [self getSerialQueue:phone];
    dispatch_async(queue, ^{
       
        int num = (int)[_peerConnectionsDict count];
        
        if (num == 1) {
            [self hangupPhone:phone];
            _tvuSignal->ClearMessageQueue();
        }else{
             [self hangupPhone:phone];
        }
    });
    
}

- (void)tvuEndUpCall:(NSArray *)phoneNumbers
{
    int callNumbers = (int)phoneNumbers.count;
    if (phoneNumbers == NULL || phoneNumbers == nil || [phoneNumbers count] <= 0) {
        log4cplus_error("WebRTCLog", "The end up call request array is null..%s",__func__);
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

    dispatch_async(TVUMainQueue, ^{
        [self.delegate endupCallWithTVUWebRTCManager:self];
    });
}

- (void)tvuAcceptCall:(NSString *)phone
{
    
    dispatch_queue_t phoneQueue = [self getSerialQueue:phone];
    dispatch_async(phoneQueue, ^{
        [self checkSwitchWebRTCSDKTimestamp];
        [self tvuSettingWebRTCSDKType:KTypeWebRTCSDK_TVU];
        [self beginCommiuncationWithPhoneNumber:phone isAccept:YES];
        
    });

}

- (void)tvuRejectCall:(NSString *)phone
{
    [self beginCommiuncationWithPhoneNumber:phone isAccept:NO];
}

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

- (void)settingPcFactoryToNil
{
    if (_pcFactory) {
        
        dispatch_queue_t phoneQueue = [self getSerialQueue:kSerialQueueTag];
        dispatch_async(phoneQueue, ^{
            while(YES){
                int count = (int)[self.peerConnectionsDict count];
                if(count) {
                    usleep(1000*1000);
                }else{
                    [_pcFactory stopAecDump];
                    _pcFactory = nil;
                    _pcFactoryIsNilTime = [TVUDateTool getCurrentTimestamp];
                    log4cplus_warn("WebRTCLog", "%s,_pcFactory=nil,line=%d",__func__,__LINE__);
                    break;
                }
            }
        });
      
    }
}

- (BOOL)isWithin5SecondsToNow
{
    if (([TVUDateTool getCurrentTimestamp] - _pcFactoryIsNilTime)<=kSwitchWebRTCSDKTime) {
        return YES;
    }
    return NO;
}

- (void)tvuSettingWebRTCSDKType:(KTypeWebRTCSDK)type
{
    switch (type) {
        case KTypeWebRTCSDK_Original:
        {
            dispatch_async(TVUGlobalQueue, ^{
                [RTCPeerConnection disableWebRTCAudioPlayer:NO];
            });
        }
            break;
        case KTypeWebRTCSDK_TVU:
        {
            dispatch_async(TVUGlobalQueue, ^{
                [RTCPeerConnection disableWebRTCAudioPlayer:YES];
            });
        }
            break;
    }
}

- (int)getWaitingTime
{
    if ([self isWithin5SecondsToNow]) {
        double timeInterval = [TVUDateTool getCurrentTimestamp] - _pcFactoryIsNilTime;
        return (int)timeInterval +1;
    }
    return 0;
}

- (void)checkSwitchWebRTCSDKTimestamp
{
    if ([self isWithin5SecondsToNow]) {
        
        [self stopSignalMessageTimer];
        double now = [TVUDateTool getCurrentTimestamp];
        log4cplus_warn("WebRTCLog", "%s,sleep=%f,after 5 seconds to do your operation..",__func__,(now - _pcFactoryIsNilTime));
        usleep(1000*1000*(now - _pcFactoryIsNilTime));
        [self startSignalMessageTimer];
    }
}

- (KTVUVoipUserState)tvuSelectCallState:(NSString *)phoneNumber
{
    if (phoneNumber == NULL || phoneNumber == nil || [phoneNumber length] <= 0) {
//        log4cplus_error("WebRTCLog", "the phoneNumber is null..%s",__func__);
        return KTVUVoipUserStateDisconnect;
    }
    log4cplus_debug("WebRTCLog", "select call state,phone=%s",[phoneNumber UTF8String]);
    NSString *senderID = [NSString stringWithFormat:@"%@%@",KAudioTrackIDPrefix,phoneNumber];
    KTVUVoipUserState userState = KTVUVoipUserStateDisconnect;
    @synchronized (self) {
        userState = (KTVUVoipUserState)[[_callStateDict objectForKey:senderID] intValue];
    }
    return userState;
}

- (void)beginCommiuncationWithPhoneNumber:(NSString *)phoneNumber isAccept:(BOOL)accept
{
    if (phoneNumber == NULL || phoneNumber == nil || [phoneNumber length] <= 0) {
        log4cplus_error("WebRTCLog", "phoneNumber is null..%s",__func__);
        return;
    }
    
    dispatch_queue_t phoneQueue = [self getSerialQueue:phoneNumber];
    if (!phoneQueue) {
        return;
    }
    dispatch_async(phoneQueue, ^{
        if (!accept) {

        }else{
            [self createPeerConnectionAndSetToContainer:phoneNumber];
            self.nowCallFromPhone = phoneNumber;
            usleep(1000*100);
        }
        bool isaccpet = accept ? true : false;
        [kTVUNotification addObserver:self selector:@selector(checkSpeakerType) name:AVAudioSessionRouteChangeNotification object:nil];
        if (phoneNumber != NULL) {
            log4cplus_warn("WebRTCLog", "%s,accept call,phone=%s",__func__,[phoneNumber UTF8String]);
            _tvuSignal->postResponse(isaccpet,[phoneNumber UTF8String]);  // accept call
        }

    });
    
}

- (NSArray *)defaultICEServers
{
    RTCIceServer *iceserver1 = [[RTCIceServer alloc] initWithURLStrings:@[kStunserver1] username:@"tvu" credential:@"tvu"];
    RTCIceServer *icerserver3 = [[RTCIceServer alloc] initWithURLStrings:@[kStunserver3]];
    RTCIceServer *icerserver4 = [[RTCIceServer alloc] initWithURLStrings:@[kStunserver4]];
    
    return @[iceserver1,icerserver3,icerserver4];
}

- (void)beginAcceptCall:(RTCPeerConnection *)peerConnection phone:(NSString *)phoneNumber
{
    if (peerConnection == NULL || peerConnection == nil) {
        log4cplus_error("WebRTCLog", "peerConnection is nil..%s",__func__);
        return;
    }
    [peerConnection answerForConstraints:nil completionHandler:^(RTCSessionDescription * _Nullable sdp, NSError * _Nullable error) {
        if (error != NULL) {
            log4cplus_error("WebRTCLog", "Create answer is failed..");
        }else{
            dispatch_async(TVUMainQueue, ^{
                [peerConnection setLocalDescription:sdp completionHandler:^(NSError * _Nullable error) {
                    if(error == NULL){
                        log4cplus_info("WebRTCLog", "Set local sdp succ..");
                        NSLog(@"WebRTC : Set local sdp succ..");
                        
                        _tvuSignal->postanswer([sdp.sdp UTF8String],[phoneNumber UTF8String]);
                        log4cplus_warn("WebRTCLog", "post answer to signal server,phone=%s",[phoneNumber UTF8String]);
                        
                    }else{
                        log4cplus_info("WebRTCLog", "Set local sdp failed..");
                        NSLog(@"WebRTC : Set local sdp failed..");
                    }
                }];
            });
        }
    }];
}

- (RTCMediaStream *)createStreamUsePhoneNumber:(NSString *)phoneNumber
{
    log4cplus_debug("WebRTCLog", "create media stream , phone=%s",[phoneNumber UTF8String]);
    RTCMediaStream *mediaStream = [self.pcFactory mediaStreamWithStreamId:[NSString stringWithFormat:@"ARDAMS%@",phoneNumber]];;
    RTCAudioTrack *audioTrack = [self.pcFactory audioTrackWithTrackId:[NSString stringWithFormat:@"%@%@",KAudioTrackIDPrefix,phoneNumber]];
    [mediaStream addAudioTrack:audioTrack];
  
    /* support video  */
    RTCVideoTrack *videoTrack = [self.pcFactory videoTrackWithSource:self.videoSource trackId:[NSString stringWithFormat:@"ARDAMSv0%@",phoneNumber]];
    [mediaStream addVideoTrack:videoTrack];
    return mediaStream;
}

- (RTCPeerConnection *)createPeerConnectionUsePhoneNumber:(NSString *)phoneNumber
{
    log4cplus_warn("WebRTCLog", "%s ,Create peerConnection, phone=%s",__func__,[phoneNumber UTF8String]);
    RTCConfiguration *config = [[RTCConfiguration alloc] init];
    config.iceTransportPolicy = RTCIceTransportPolicyAll;
    [config setIceServers:[self defaultICEServers]];
    RTCPeerConnection *peerConnection = [self.pcFactory peerConnectionWithConfiguration:config constraints:nil delegate:self];
    [peerConnection addStream:[self createStreamUsePhoneNumber:phoneNumber]];
    return peerConnection;
}

- (RTCPeerConnection *)getNewestPeerConnection
{
    if (self.nowCallFromPhone == NULL || self.nowCallFromPhone == nil) {
//        log4cplus_error("WebRTCLog", "Get newest peerConnection failed..");
        return NULL;
    }
    TVULOCK(_tvuWebRTCLock);
    TVUWebRTCPeerConnectionDescription *peerConnDes = [_peerConnectionsDict objectForKey:self.nowCallFromPhone];
    TVUUNLOCK(_tvuWebRTCLock);
    return peerConnDes.peerConnection;
}

- (BOOL)voipIsCalling
{
    return [self getNewestPeerConnection] != NULL;
}

- (void)tvuWriteAudiodataToWebRTC:(int8_t *)data dataSize:(size_t)size writeTime:(float)writeTime
{
    if (data == NULL || size <= 0 ) {
        log4cplus_error("WebRTCLog", "write audio data to WebRTC failed, audio data is null...");
        return;
    }
    if (!self.isEndupCall) {
        dispatch_queue_t phoneQueue = [self getSerialQueue:kSerialQueueTag];
        dispatch_async(phoneQueue, ^{
            RTCPeerConnection *rtcPeerConn = [self getNewestPeerConnection];
            if (rtcPeerConn) {
                [rtcPeerConn acceptRecodedAudioData:data size:size writeTime:writeTime];
//                log4cplus_warn("WebRTCLogAudio", "write audio data to webrtc..");
            }
        });
    }
}

- (RTCPeerConnection *)getPeerConnectionUsePhoneNumber:(NSString *)phoneNumber
{
    if (phoneNumber == NULL || phoneNumber == nil) {
        log4cplus_error("WebRTCLog", "The phoneNumber is null..");
        return NULL;
    }
    TVULOCK(_tvuWebRTCLock);
    TVUWebRTCPeerConnectionDescription *peerConnDes = [_peerConnectionsDict objectForKey:phoneNumber];
    TVUUNLOCK(_tvuWebRTCLock);
    return peerConnDes.peerConnection;
}

- (void)createPeerConnectionAndSetToContainer:(NSString *)phoneNumber
{
    RTCPeerConnection *peerConnection = NULL;
    TVUWebRTCPeerConnectionDescription *peerConnDes = NULL;
    peerConnection = [self createPeerConnectionUsePhoneNumber:phoneNumber];
    peerConnDes = [TVUWebRTCPeerConnectionDescription tvuWebRTCPeerConnectionDescriptionWithDict:@{@"peerConnection":peerConnection,@"phoneNumber":phoneNumber}];
    TVULOCK(_tvuWebRTCLock);
    [_peerConnectionsDict setObject:peerConnDes forKey:phoneNumber];
    TVUUNLOCK(_tvuWebRTCLock);
}

- (void)removeElementsFromPeerConnectionsDict:(NSString *)phoneNumber andType:(KRemovePeerConnectionType)type
{
    log4cplus_warn("WebRTCLog", "Remove the peerConnection %s",[phoneNumber UTF8String]);
    TVULOCK(_tvuWebRTCLock);
    if ([_peerConnectionsDict objectForKey:phoneNumber] != NULL) {
        [_peerConnectionsDict removeObjectForKey:phoneNumber];
    }
    TVUUNLOCK(_tvuWebRTCLock);
    
    if (type == KRemovePeerConnectionTypeDisconnect) {
        NSMutableDictionary *message = [NSMutableDictionary dictionaryWithObject:[NSString stringWithFormat:@"End of TVU Voice from %@",phoneNumber] forKey:@"Data"];
        [kTVUNotification postNotificationName:kDistributeMsg object:nil userInfo:message];
    }
}

- (void)processingMessageQueue
{
    TVUVOIPMessageQueue *messageQueue = _tvuSignal->m_messageQueue;
    
    VOIPQNode *node = _tvuSignal->DeQueue(messageQueue);
    if (node == NULL || node == nil) {
        return;
    }
    KSignalingType messtype = node->type;
    char* message = node->data;
    NSString *messageStr = [NSString stringWithUTF8String:message];
//    NSLog(@"---%s----%@---%d",__func__,messageStr,messtype);

    if (messageStr.length <= 0 || messageStr == NULL || messageStr == nil) {
        return;
    }
    
    if (messtype == KSignalingTypeLogin ) {
        [self processLoginInfoUseMessageData:messageStr];
        return;
    }
    
    if ([messageStr isEqualToString:@"{}"]) {
        log4cplus_error("WebRTCLog", "%s,Return info  is null..",__func__);
        return;
    }
    
    NSString *callFromNumber = [NSJSONSerialization getJsonValueWithKey:@"from" jsonString:messageStr];
    if (callFromNumber == NULL || callFromNumber == nil) {
        log4cplus_error("WebRTCLog", "%s,phone number is null..",__func__);
        return;
    }
    
    if (messtype == KSignalingTypeOffer || messtype == KSignalingTypeIce || messtype == KSignalingTypeCallResponse || messtype == KSignalingTypeAnswer) {
        
        if ([self shouldDismissThisSignal:callFromNumber]) {
            return;
        }
        
    }
    
    switch (messtype) {
        case KSignalingTypeOffer:
        {
            log4cplus_warn("WebRTCLog", "%s,receive offer info, phone=%s",__func__,[callFromNumber UTF8String]);
            
            [self processOfferInfoUseMessageData:messageStr andPhoneNumber:callFromNumber];
        }
            break;
        case KSignalingTypeIce:
        {
            log4cplus_warn("WebRTCLog", "%s,receive ice info, phone=%s",__func__,[callFromNumber UTF8String]);
            
            [self processIceCandidateUseMessageData:messageStr andPhoneNumber:callFromNumber];
        }
            break;
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
        case KSignalingTypeAnswer:
        {
            log4cplus_warn("WebRTCLog", "%s,receive answer info, phone=%s",__func__,[callFromNumber UTF8String]);
            
            [self processAnswerUseMessageData:messageStr andPhoneNumber:callFromNumber];
        }
            break;
        case KSignalingTypeCallResponse:
        {
            log4cplus_warn("WebRTCLog", "%s,receive callResponse info, phone=%s",__func__,[callFromNumber UTF8String]);
            
            [self processCallResponseUseMessageData:messageStr andPhoneNumber:callFromNumber];
        }
            break;
        case KSignalingTypeDisconnectPeer:
        {
            log4cplus_warn("WebRTCLog", "%s,receive disconnect info, phone=%s",__func__,[callFromNumber UTF8String]);
            
            dispatch_queue_t phoneQueue = [self getSerialQueue:callFromNumber];
            dispatch_async(phoneQueue, ^{
                [self updateSignalingStateWithPhone:callFromNumber andState:KTVUVoiceState_disconnect];     // set phone state is disconnect
            });
            
            [self processDisconnectPeerUseMessageData:messageStr andPhoneNumber:callFromNumber];
        }
            break;
            
        default:
            break;
    }
    _tvuSignal->FreeNode(node);
    
}

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
//        [self.delegate endupCallWithTVUWebRTCManager:self];
    });

}

- (void)processCallRequestUseMessageData:(NSString *)message andPhoneNumber:(NSString *)phoneNumber
{

//    self.nowCallFromPhone = callFromNumber;
//    dispatch_async(TVUMainQueue, ^{
//        [self.delegate acceptCallWithTVUWebRTCManager:self PhoneNumber:phoneNumber];
//    });
    
    [self tvuAcceptCall:phoneNumber];
}

- (void)processCallResponseUseMessageData:(NSString *)message andPhoneNumber:(NSString *)phoneNumber
{

    int status = [[NSJSONSerialization getJsonValueWithKey:@"response" jsonString:message] intValue];
    
    if (status == 0) {
        log4cplus_error("WebRTCLog", "WebRTC CallResponse: The server returned an error,the user may not exist..");
        return;
    }

    [self createPeerConnectionAndSetToContainer:phoneNumber];
    
    __block RTCPeerConnection *peerConnection = NULL;
    dispatch_queue_t phoneQueue = [self getSerialQueue:phoneNumber];
    dispatch_async(phoneQueue, ^{
        peerConnection = [self getPeerConnectionUsePhoneNumber:phoneNumber];
        [peerConnection offerForConstraints:nil completionHandler:^(RTCSessionDescription * _Nullable sdp, NSError * _Nullable error) {
            if (sdp == NULL || sdp == nil) {
                log4cplus_error("WebRTCLog", "WebRTC CallResponse: create offer is failed(sdp info is null..)..");
                return;
            }
            log4cplus_warn("WebRTCLog", "%s,post offer to signal server,phone=%s",__func__,[phoneNumber UTF8String]);
            _tvuSignal->postoffer([sdp.sdp UTF8String], [phoneNumber UTF8String]);
            [peerConnection setLocalDescription:sdp completionHandler:^(NSError * _Nullable error) {
                if (error == NULL){
                    log4cplus_info("WebRTCLog", "Set local sdp succ..");
                }else{
                    log4cplus_error("WebRTCLog", "Set local sdp failed..");
                }
            }];
        }];

    });
    
}

- (void)processAnswerUseMessageData:(NSString *)message andPhoneNumber:(NSString *)phoneNumber
{
    dispatch_queue_t phoneQueue = [self getSerialQueue:phoneNumber];
    __block RTCPeerConnection *peerConnection = NULL;
    dispatch_async(phoneQueue, ^{
        
        peerConnection = [self getPeerConnectionUsePhoneNumber:phoneNumber];
        if (!peerConnection) {
            log4cplus_error("WebRTCLog", "%s,line=%d,peerConn=null...",__func__,__LINE__);
            return;
        }
        NSString *sdpstr = [NSJSONSerialization getJsonValueWithKey:@"sdp" jsonString:message];
        if (sdpstr != NULL) {
            RTCSessionDescription *remoteSDP = [[RTCSessionDescription alloc] initWithType:RTCSdpTypePrAnswer sdp:sdpstr];
            [peerConnection setRemoteDescription:remoteSDP completionHandler:^(NSError * _Nullable error) {
                if (error == NULL){
                    log4cplus_info("WebRTCLog", "set remote sdp succ,phone=%s",[phoneNumber UTF8String]);
//                    NSLog(@"WebRTC : set remote sdp succ..");
                }else{
                    log4cplus_info("WebRTCLog", "set remote sdp failed,phone=%s",[phoneNumber UTF8String]);
//                    NSLog(@"WebRTC : set remote sdp failed..");
                }
            }];
        }
    });
    
}

- (void)processLoginInfoUseMessageData:(NSString *)message
{
    if ([message isEqualToString:@"1" ]) {
        log4cplus_debug("WebRTCLog", "Login in success..");
    }
}

- (void)processOfferInfoUseMessageData:(NSString *)message andPhoneNumber:(NSString *)phoneNumber
{
    dispatch_queue_t phoneQueue = [self getSerialQueue:phoneNumber];
    dispatch_async(phoneQueue, ^{
        
        RTCPeerConnection *peerConnection = NULL;
        peerConnection = [self getPeerConnectionUsePhoneNumber:phoneNumber];
        if (peerConnection == NULL || peerConnection == nil) {
            log4cplus_error("WebRTCLog", "%s,line=%d,peerConn=null...",__func__,__LINE__);
            return;
        }
        NSData *jsonData = [message dataUsingEncoding:NSUTF8StringEncoding];
        NSDictionary *dic = [NSJSONSerialization JSONObjectWithData:jsonData options:NSJSONReadingMutableContainers error:NULL];
        NSString *sdpstr = [dic objectForKey:@"sdp"];
        
        RTCSessionDescription *remoteSDP = [[RTCSessionDescription alloc] initWithType:RTCSdpTypeOffer sdp:sdpstr];
        if (remoteSDP != NULL) {
            [peerConnection setRemoteDescription:remoteSDP completionHandler:^(NSError * _Nullable error) {
                if (error == NULL){
                    log4cplus_info("WebRTCLog", "set remote sdp succ..");
                    NSLog(@"set remote sdp succ");
                }
                else{
                    log4cplus_info("WebRTCLog", "set remote sdp failed..");
                    NSLog(@"set remote sdp failed");
                }
            }];
        }
        [self beginAcceptCall:peerConnection phone:phoneNumber];
    });
}

- (void)processIceCandidateUseMessageData:(NSString *)message andPhoneNumber:(NSString *)phoneNumber
{
    NSString *candidate = [NSJSONSerialization getJsonValueWithKey:@"candidate" jsonString:message];
    NSString *sdpMLineIndexStr = [NSJSONSerialization getJsonValueWithKey:@"sdpMLineIndex" jsonString:message];
    NSString *sdpMid = [NSJSONSerialization getJsonValueWithKey:@"sdpMid" jsonString:message];
    RTCIceCandidate *iceCandidate = [[RTCIceCandidate alloc] initWithSdp:candidate sdpMLineIndex:[sdpMLineIndexStr intValue] sdpMid:sdpMid];
    
    dispatch_queue_t phoneQueue = [self getSerialQueue:phoneNumber];
    dispatch_async(phoneQueue, ^{
        
//        RTCPeerConnection *peerConnection = NULL;
       RTCPeerConnection *peerConnection = [self getPeerConnectionUsePhoneNumber:phoneNumber];
        if (!peerConnection) {
            log4cplus_error("WebRTCLog", "%s,line=%d,peerConn=null...",__func__,__LINE__);
            return;
        }
        [peerConnection addIceCandidate:iceCandidate];
    });
}

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

- (void)checkSpeakerType
{
    AVAudioSessionRouteDescription *audioSessionRoteDescription = [[AVAudioSession sharedInstance] currentRoute];
    
    NSArray *audioSessionPortDesArray  = [audioSessionRoteDescription outputs];
    
    for (AVAudioSessionPortDescription  *audioSessionPortDescription in audioSessionPortDesArray) {
        NSString *outputPortType = audioSessionPortDescription.portType;
        
        if ([outputPortType isEqualToString:@"Speaker"]) {

            break;
        }
        if ([outputPortType isEqualToString:@"Headphones"]) {

            break;
        }
    }
}

#pragma mark -RTCPeerConnectionDelegate
/** Called when the SignalingState changed. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didChangeSignalingState:(RTCSignalingState)stateChanged
{
   
}

/** Called when media is received on a new stream from remote peer. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
          didAddStream:(RTCMediaStream *)stream
{
    NSString *phone = [self getCallerUsePeerConnection:peerConnection];
    dispatch_queue_t queue = [self getSerialQueue:phone];
    dispatch_async(queue, ^{
        RTCVideoTrack *videoTracker = [stream.videoTracks objectAtIndex:0];
        [self.delegate receiveRemoteVideoStream:videoTracker phoneNumber:phone];
    });
}

/** Called when a remote peer closes a stream. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
       didRemoveStream:(RTCMediaStream *)stream
{
 
}

/** Called when negotiation is needed, for example ICE has restarted. */
- (void)peerConnectionShouldNegotiate:(RTCPeerConnection *)peerConnection
{
 
}

/** Called any time the IceConnectionState changes. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didChangeIceConnectionState:(RTCIceConnectionState)newState
{
    log4cplus_warn("WebRTCLog", "%s,newState=%d,line=%d",__func__,(int)newState,__LINE__);
//    if (newState == RTCIceConnectionStateClosed) {
//        log4cplus_warn("WebRTCLog", "%s,line=%d,newState is closed...",__func__,__LINE__);
//        return;
//    }
    
    NSArray *peerSenders = peerConnection.senders;
    if (!peerSenders || [peerSenders count] <= 0) {
        log4cplus_warn("WebRTCLog", "%s,newState=%d,line=%d",__func__,(int)newState,__LINE__);
        return;
    }
    RTCRtpSender *rtcSender = [peerSenders objectAtIndex:0];
    NSString *senderID =rtcSender.senderId;
    if (!senderID) {
        log4cplus_warn("WebRTCLog", "%s,line=%d,phone=null",__func__,__LINE__);
        return;
    }
    
    KTVUVoipUserState userState = KTVUVoipUserStateDisconnect;
    self.isEndupCall = YES;
    switch (newState) {
        case RTCIceConnectionStateNew:
        case RTCIceConnectionStateChecking:
            userState = KTVUVoipUserStateConnecting;
            break;
        case RTCIceConnectionStateConnected:
        case RTCIceConnectionStateCompleted:
        {
            userState = KTVUVoipUserStateCalling;
            self.isEndupCall = NO;
            log4cplus_warn("WebRTCLogAudio", "calling now, can write audio data to webrtc..");
        }
            break;
        default:
            break;
    }
    @synchronized (self) {
        [_callStateDict setObject:[NSNumber numberWithInt:userState] forKey:senderID];
    }
}

/** Called any time the IceGatheringState changes. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didChangeIceGatheringState:(RTCIceGatheringState)newState
{
//    log4cplus_warn("WebRTCLog", "%s,newState=%d,line=%d",__func__,(int)newState,__LINE__);
//    if (newState == RTCIceGatheringStateComplete) {
//        NSString *phone = [self getCallerUsePeerConnection:peerConnection];
//         log4cplus_warn("WebRTCLog", "%s,newState=%d,phone=%s,line=%d",__func__,(int)newState,[phone UTF8String],__LINE__);
//        if (!phone) {
//            log4cplus_warn("WebRTCLog", "%s,line=%d,phone=null",__func__,__LINE__);
//            return;
//        }
//        _tvuSignal->postanswer([peerConnection.localDescription.sdp UTF8String],[phone UTF8String]);
//        log4cplus_warn("WebRTCLog", "post answer to signal server,phone=%s",[phone UTF8String]);
//    }
}

/** New ice candidate has been found. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didGenerateIceCandidate:(RTCIceCandidate *)candidate
{
    NSString *phone = [self getCallerUsePeerConnection:peerConnection];
    dispatch_queue_t queue = [self getSerialQueue:phone];
    
    log4cplus_warn("WebRTCLog", "%s,phone=%s,line=%d",__func__,[phone UTF8String],__LINE__);
    if (!phone) {
        log4cplus_warn("WebRTCLog", "%s,line=%d,phone=null",__func__,__LINE__);
        return;
    }
    dispatch_async(queue, ^{
//        log4cplus_warn("WebRTCLog", "%s,line=%d",__func__,__LINE__);
        NSData *data = [candidate JSONData];
        NSDictionary *dict = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingMutableLeaves error:NULL];
        NSString *candidateStr = (NSString *)[dict objectForKey:@"candidate"];
        
        _tvuSignal->postice([candidateStr UTF8String], [candidate.sdpMid UTF8String], [[NSString stringWithFormat:@"%ld",(long)candidate.sdpMLineIndex] UTF8String],[phone UTF8String]);
        log4cplus_warn("WebRTCLog", "post ice to signal server,phone=%s",[phone UTF8String]);
    });
    
    

}

/** Called when a group of local Ice candidates have been removed. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
didRemoveIceCandidates:(NSArray<RTCIceCandidate *> *)candidates
{

}

/** New data channel has been opened. */
- (void)peerConnection:(RTCPeerConnection *)peerConnection
    didOpenDataChannel:(RTCDataChannel *)dataChannel
{

}

#pragma mark --qi define
- (NSString *)getCallerUsePeerConnection:(RTCPeerConnection *)peerConnection
{
    if (!peerConnection) {
        return  NULL;
    }
    NSArray *peerSenders = peerConnection.senders;
    if (!peerSenders || [peerSenders count] <= 0) {
        return NULL;
    }
    
    RTCRtpSender *rtcSender = [peerSenders objectAtIndex:0];
    NSString *senderID = rtcSender.senderId;
    if (senderID == NULL || [senderID length]<=0) {
        return NULL;
    }
    return [senderID substringFromIndex:[KAudioTrackIDPrefix length]];
}


#pragma mark --function test

///************* call/endup call  *****************/
//- (void)testCallAndEndupFunc
//{
//    [NSTimer scheduledTimerWithTimeInterval:10.0 target:self selector:@selector(repeatCallAndEndup) userInfo:nil repeats:YES];  // test 3、10、20 seconds
//    [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(repeatSelectPhoneState) userInfo:nil repeats:YES];
//}
//
//- (void)repeatCallAndEndup
//{
//    static int callCount = 1;
//    if (callCount%2 != 0) {
//        NSLog(@"begin call...");
//        [self tvuCallRequest:@[@"112233",@"223344",@"334455",@"445566",@"556677",@"667788",@"778899"]];
//    }else{
//        [self tvuEndUpCall:@[@"112233",@"223344",@"334455",@"445566",@"556677",@"667788",@"778899"]];
//        NSLog(@"endup call...");
//    }
//    callCount++;
//    if (callCount == 7) {
//        callCount = 1;
//    }
//    if (callCount == 3) {
//        [self tvuSettingWebRTCSDKType:KTypeWebRTCSDK_TVU];
//    }
//    if (callCount == 5) {
//        [self tvuSettingWebRTCSDKType:KTypeWebRTCSDK_Original];
//    }
//}
//
///************* select call state  *****************/
//- (void)repeatSelectPhoneState
//{
//    NSArray *array = @[@"112233",@"223344",@"334455",@"445566",@"556677",@"667788",@"778899"];
//    for (NSString *number in array) {
//        NSLog(@"state--%@=%d",number,(int)[self tvuSelectCallState:number]);
//    }
//}
//
//
///*************  test hangupcall   *****************************/
//- (void)testHangupcall
//{
//    [NSTimer scheduledTimerWithTimeInterval:30.0 target:self selector:@selector(repeatCallAndHangup) userInfo:nil repeats:YES];
//}
//
//- (void)repeatCallAndHangup
//{
//    static int call_Count = 1;
//    if (call_Count%2 != 0) {
//        NSLog(@"begin call...");
//        [self tvuCallRequest:@[@"112233",@"223344",@"334455",@"445566",@"556677",@"667788",@"778899"]];
//    }else{
//        NSArray *array = @[@"112233",@"223344",@"334455",@"445566",@"556677",@"667788",@"778899"];
//        for (NSString *phone in array) {
//            [self tvuHangupCall:phone];
//            NSLog(@"endup call...");
//        }
//    }
//    call_Count++;
//    if (call_Count == 7) {
//        call_Count = 1;
//    }
//}

@end

```