---
layout: post
title:  AURemoteIO中的IOThread崩溃
description: VOIP模块，在挂断电话时发生了闪退
category: project
tag: viop,webrtc,AURemoteIO,IOThread
---
## 场景说明

在TVUCC界面做voip通话时，点击返回按钮挂断电话并返回到主页面时发生crash。以下是crash的堆栈信息。

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20170829AURemoteIO/IOThread02.png)

## bug分析

根据以上堆栈信息以及crash发生的步骤，我们得出一下信息：

1. `AURemoteIO::IOThread`是`audio unit`的工作线程
2.  crash的发生点是在`webrtc`中，并且是`webrtc`应用`audio unit`的模块发生了问题，有可能是音频的播放，也有可能是音频录制。
3.  根据crash的发生步骤，是在挂断电话回到主页面时发生的，那么这一操作底层做的事情有：
	* 挂断电话，包含`[peerConn close]`,'_peerConn = nil',`_pcFactory = nil`
	* 切换界面，包含SDK的切换

根据以上信息，我们推测：发生crash的点很有可能和`peerConnection`,`pcFactory`,SDK切换有直接的关系，并且是在`audio unit`停止时发生的问题。那么我们就通过分析`webrtc`源码结合我们应用的代码来查找出`audio unit`是什么时机停的，我们当前的挂断电话和切换sdk逻辑是否正确。

`webrtc`中停止`audio unit`的逻辑：

```
bool VoiceProcessingAudioUnit::Stop() {
  RTC_DCHECK_GE(state_, kUninitialized);
  RTCLog(@"Stopping audio unit.");
  if (!mDisableWebRTCAudioPlayer)
  {
     OSStatus result = AudioOutputUnitStop(vpio_unit_);
    if (result != noErr) {
      RTCLogError(@"Failed to stop audio unit. Error=%ld", (long)result);
      return false;
    } else {
      RTCLog(@"Stopped audio unit");
    }
    RTCLog(@"%s,use original WebRTC...",__func__);
  }else{
    RTCLog(@"%s,use TVUWebRTC...",__func__);
  }


  state_ = kInitialized;
  return true;
}
```

## 测试用例

### case1

步骤：(QA:10.12.128.155)

1) 进入TVUCC界面，拨打多路电话，只有1路能拨通

2）点击返回按钮挂断电话并回到主页面

logs:

```
2017-08-30 09:47:45.231313+0800 TVUAnywhere[9594:2441185] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],hang up phone,phone=0x50C700408BE7CC44
2017-08-30 09:47:45.231430+0800 TVUAnywhere[9594:2441185] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----lock-----474
2017-08-30 09:47:45.231447+0800 TVUAnywhere[9594:2441185] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----unlock-----476
2017-08-30 09:47:45.231469+0800 TVUAnywhere[9594:2441185] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],remove mediaStream,phone=0x50C700408BE7CC44,line=249
(port.cc:1404): Jingle:Conn[0x160227800:audio:JCpeBETR:1:0:local:udp:10.216.132.x:55447->T5V9qmoc:1:2122260223:local:udp:192.168.1.x:61243|C--I|0|0|9114475305677766142|-]: Timing-out STUN ping 7571334468566d7036424156 after 5003 ms
(port.cc:1161): Jingle:Conn[0x160a35600:audio:1DhhzuSn:1:0:local:udp:192.168.1.x:49891->T5V9qmoc:1:2122260223:local:udp:192.168.1.x:61243|CRWS|0|0|9114756780654476798|437]: UpdateState(), ms since last received response=1154, ms since last received data=0, rtt=874, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x1602fd600:audio:RyDoC+oe:1:0:local:udp:169.254.66.x:52065->T5V9qmoc:1:2122260223:local:udp:192.168.1.x:61243|C--I|0|0|9115038255631187454|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=372b3057354e49745370746a 33724e6f6970544346784272 4a4969747050372b394b7242 686d77575a496859306c4d6e 6c2f774a565563787a44704a ... 19 more
(port.cc:1161): Jingle:Conn[0x160331400:audio:RyDoC+oe:1:0:local:udp:169.254.66.x:52065->2y7DdHPW:1:2122194687:local:udp:10.12.128.x:41777|C--I|0|0|9114756780654476799|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=796d7a4a4c65386d494e3371 34704d4647375952416b5842 57534f30717a573834577334 714d4630784532566b2b7a71 566a3069545345444557344a ... 19 more
(port.cc:1161): Jingle:Conn[0x160a3c400:audio:Vrts555D:1:0:local:tcp:169.254.66.x:54206->5O+U9nog:1:1518280447:local:tcp:192.168.1.x:11674|C--I|0|0|6520964869057822206|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=396b3164625448784e515966 366c7058506e467973525471 6679354a3969632b4265636b 415a644e6a7248734e49396c 6174432f425868725644595a ... 19 more
(port.cc:1161): Jingle:Conn[0x160a4b800:audio:Vrts555D:1:0:local:tcp:169.254.66.x:54206->3yamMqI+:1:1518214911:local:tcp:10.12.128.x:33318|C--I|0|0|6520683394081111551|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=6b37735231755449516a624b 314b756247452b67387a5548 357233484861674b39792b51 2b396c765338505951323078 344d6a2f6a30584558667173 ... 18 more
(port.cc:1161): Jingle:Conn[0x160227800:audio:JCpeBETR:1:0:local:udp:10.216.132.x:55447->T5V9qmoc:1:2122260223:local:udp:192.168.1.x:61243|C--I|0|0|9114475305677766142|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=4d56544d623446494f574942 4d384c73794a4c2b2f684638 6b47365a6f6d4d484e654876 4f56724b7144795442394f4d 7255784d38586a39716a7533 ... 18 more
(port.cc:1161): Jingle:Conn[0x160330c00:audio:JCpeBETR:1:0:local:udp:10.216.132.x:55447->2y7DdHPW:1:2122194687:local:udp:10.12.128.x:41777|C--I|0|0|9114475305677635070|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=4c517030735066377555506e 6f485736627a7a7038393668 633273663059717879747061 3657546a6b3271334f4f694a 486e4377675478336a574d39 ... 18 more
(port.cc:1161): Jingle:Conn[0x160a22a00:audio:c7eCZBiS:1:0:local:tcp:10.216.132.x:54208->5O+U9nog:1:1518280447:local:tcp:192.168.1.x:11674|---W|0|0|6520401919104400894|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x160a3bc00:audio:c7eCZBiS:1:0:local:tcp:10.216.132.x:54208->3yamMqI+:1:1518214911:local:tcp:10.12.128.x:33318|---W|0|0|6520401919104269822|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x160a38c00:audio:AnT1kMOS:1:0:local:tcp:192.168.1.x:54207->oVbnpAl/:1:1350508287:prflx:tcp:192.168.1.x:34383|CRxS|0|0|5800388928678411775|2297]: UpdateState(), ms since last received response=988, ms since last received data=528763046, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x160a44400:audio:1DhhzuSn:1:0:local:udp:192.168.1.x:49891->2y7DdHPW:1:2122194687:local:udp:10.12.128.x:41777|C-xW|0|0|9114756780654345726|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x160331c00:audio:AnT1kMOS:1:0:local:tcp:192.168.1.x:54207->5O+U9nog:1:1518280447:local:tcp:192.168.1.x:11674|C-xW|0|0|6520683394081111550|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x160a4b000:audio:AnT1kMOS:1:0:local:tcp:192.168.1.x:54207->3yamMqI+:1:1518214911:local:tcp:10.12.128.x:33318|C-xW|0|0|6520683394080980478|-]: UpdateState(), ms since last received response=528763046, ms since last received data=528763046, rtt=3000, pings_since_last_response=
(port.cc:1222): Jingle:Conn[0x160a4b800:audio:Vrts555D:1:0:local:tcp:169.254.66.x:54206->3yamMqI+:1:1518214911:local:tcp:10.12.128.x:33318|C--I|0|0|6520683394081111551|-]: Sending STUN ping , id=615371715575445843492f30, nomination=0
(tcpport.cc:215): Jingle:Port[0x161d16d50:audio:1:0:local:Net[en3:169.254.0.x/16:Wifi]]: Attempted to send to an unknown destination, 10.12.128.x:33318
(port.cc:966): Jingle:Conn[0x160a4b800:audio:Vrts555D:1:0:local:tcp:169.254.66.x:54206->3yamMqI+:1:1518214911:local:tcp:10.12.128.x:33318|C--I|0|0|6520683394081111551|-]: Failed to send STUN ping  err=-1 id=615371715575445843492f30
(port.cc:1412): Jingle:Conn[0x160a4b800:audio:Vrts555D:1:0:local:tcp:169.254.66.x:54206->3yamMqI+:1:1518214911:local:tcp:10.12.128.x:33318|C--I|0|0|6520683394081111551|-]: Sent STUN ping, id=615371715575445843492f30, use_candidate=0, nomination=0
(webrtcsession.cc:820): Session:6431744067696756918 Old state:STATE_RECEIVEDPRANSWER New state:STATE_CLOSED
2017-08-30 09:47:45.246774+0800 TVUAnywhere[9594:2441652] "WebRTCLog":-[TVUWebRTCManager peerConnection:didChangeIceConnectionState:],newState=6,line=858
(RTCLogging.mm:31): (RTCRtpSender.mm:89 -[RTCRtpSender initWithNativeRtpSender:]): RTCRtpSender(0x17401af10): created sender: RTCRtpSender {
  senderId: ARDAMSa00x50C700408BE7CC44
}
(webrtcvoiceengine.cc:2352): SetOutputVolume() to 0 for recv stream with ssrc 2611350844
(channel.cc:882): Channel disabled
(webrtcvoiceengine.cc:1425): Stopping playout for channel #1
(audio_device_impl.cc:1406): StopPlayout
(audio_device_ios.mm:228): AudioDeviceIOS::StopPlayout
(audio_device_impl.cc:1409): output: 0
(bitrate_allocator.cc:171): UpdateAllocationLimits : total_requested_min_bitrate: 0bps, total_requested_padding_bitrate: 0bps
(audio_device_impl.cc:1446): StopRecording
(audio_device_ios.mm:267): AudioDeviceIOS::StopRecording
(audio_device_ios.mm:934): AudioDeviceIOS::ShutdownPlayOrRecord
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:272 Stop): Stopping audio unit.
(opensslstreamadapter.cc:703): OpenSSLStreamAdapter::OnEvent SE_READ
(opensslstreamadapter.cc:721):  -- onStreamReadable
(opensslstreamadapter.cc:564): OpenSSLStreamAdapter::Read(2048)
(opensslstreamadapter.cc:626):  -- remote side closed
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:280 Stop): Stopped audio unit
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:282 Stop): Stop,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:294 Uninitialize): Unintializing audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:303 Uninitialize): Uninitialized audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:305 Uninitialize): Uninitialize,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:413 DisposeAudioUnit): Disposing audio unit.
```

`audio unit`能成功停止。


### case2

步骤：(QA:10.12.128.155)

1) 进入TVUCC界面，拨打多路电话，只有1路能拨通

2）在tvucc界面挂断这路电话，不反回主页面

logs:

```
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:272 Stop): Stopping audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:280 Stop): Stopped audio unit
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:282 Stop): Stop,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:294 Uninitialize): Unintializing audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:303 Uninitialize): Uninitialized audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:305 Uninitialize): Uninitialize,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:413 DisposeAudioUnit): Disposing audio unit.
```

`audio unit`能正常停止。

### case3

步骤：(QA:10.12.128.155)

1) 进入TVUCC界面，拨打多路电话，只有2路能拨通

2）先在TVUCC界面挂断一路

3）点击返回按钮挂断电话并回到主页面

logs:

```
(audio_device_ios.mm:934): AudioDeviceIOS::ShutdownPlayOrRecord
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:272 Stop): Stopping audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:280 Stop): Stopped audio unit
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:282 Stop): Stop,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:294 Uninitialize): Unintializing audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:303 Uninitialize): Uninitialized audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:305 Uninitialize): Uninitialize,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:413 DisposeAudioUnit): Disposing audio unit.
```

`audio unit`能成功停止。

### case4

步骤：(RD:10.12.23.232)

1) 进入TVUCC界面，打通1路电话

2）点击返回按钮挂断电话并回到主页面

logs:

```
2017-08-30 09:30:47.733525+0800 TVUAnywhere[9555:2434886] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],hang up phone,phone=112233
2017-08-30 09:30:47.733653+0800 TVUAnywhere[9555:2434886] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----lock-----475
2017-08-30 09:30:47.733667+0800 TVUAnywhere[9555:2434886] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----unlock-----477
2017-08-30 09:30:47.733684+0800 TVUAnywhere[9555:2434886] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],remove mediaStream,phone=112233,line=249
(port.cc:1161): Jingle:Conn[0x1049c0000:audio:Whv0NFgW:1:0:local:udp:169.254.66.x:62374->CImf/abu:1:2122260223:local:udp:169.254.37.x:60000|CRWS|0|0|9115038255631187454|401]: UpdateState(), ms since last received response=765, ms since last received data=12, rtt=802, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x104284800:audio:4hOcykh7:1:0:prflx:udp:10.12.24.x:53752->G4dioeWE:1:2122194687:local:udp:10.12.23.x:54559|C-WS|0|0|7961835276047498750|732]: UpdateState(), ms since last received response=1898, ms since last received data=527745386, rtt=1464, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x104a34200:audio:gpl03god:1:0:local:udp:192.168.1.x:53752->CImf/abu:1:2122260223:local:udp:169.254.37.x:60000|C--I|0|0|9114756780654476798|-]: UpdateState(), ms since last received response=527745386, ms since last received data=527745386, rtt=3000, pings_since_last_response=544d734b6b2f662f6f452b6e 61356653683761635456437a 684d55783644705071375a77 4f2f496f3956672f5a744668 6b3074464530416449574c7a 
(port.cc:1161): Jingle:Conn[0x104a39c00:audio:fQu49BvT:1:0:local:udp:10.216.132.x:54965->CImf/abu:1:2122260223:local:udp:169.254.37.x:60000|C--I|0|0|9114475305677766142|-]: UpdateState(), ms since last received response=527745386, ms since last received data=527745386, rtt=3000, pings_since_last_response=506a4e56384a30576f437769 4f7351475567326174335669 4c6b714433344b512f686842 71584d2b54434d6d67453639 4c535856714e417746326957 
(port.cc:1161): Jingle:Conn[0x1042b1600:audio:fQu49BvT:1:0:local:udp:10.216.132.x:54965->G4dioeWE:1:2122194687:local:udp:10.12.23.x:54559|C--I|0|0|9114475305677635070|-]: UpdateState(), ms since last received response=527745386, ms since last received data=527745386, rtt=3000, pings_since_last_response=4b3356595050766d796a5037 456a674771544e7647427966 6551657a65576d6c6b72345a 6c6a714a5049767232597535 4442414766385a2f442b4758 
(port.cc:1161): Jingle:Conn[0x104a39400:audio:fQu49BvT:1:0:local:udp:10.216.132.x:54965->XQs2gboZ:1:2122129151:local:udp:192.168.3.x:56696|C--I|0|0|9114475305677503998|-]: UpdateState(), ms since last received response=527745386, ms since last received data=527745386, rtt=3000, pings_since_last_response=76613656777272617a714c64 32416c79596a3636496a785a 5a592f516d3070367077452b 6433334e464e4c6e447a4178 
(port.cc:1161): Jingle:Conn[0x10496fc00:audio:Whv0NFgW:1:0:local:udp:169.254.66.x:62374->XQs2gboZ:1:2122129151:local:udp:192.168.3.x:56696|CRxW|0|0|9114475305677766143|-]: UpdateState(), ms since last received response=527745386, ms since last received data=527745386, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x104a38c00:audio:Whv0NFgW:1:0:local:udp:169.254.66.x:62374->G4dioeWE:1:2122194687:local:udp:10.12.23.x:54559|C-xW|0|0|9114756780654476799|-]: UpdateState(), ms since last received response=527745386, ms since last received data=527745386, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x104285000:audio:gpl03god:1:0:local:udp:192.168.1.x:53752->XQs2gboZ:1:2122129151:local:udp:192.168.3.x:56696|C-xI|0|0|9114475305677635071|-]: UpdateState(), ms since last received response=527745386, ms since last received data=527745386, rtt=3000, pings_since_last_response=623477706263625a6561662f 
(webrtcsession.cc:820): Session:2560792759188315450 Old state:STATE_RECEIVEDPRANSWER New state:STATE_CLOSED
2017-08-30 09:30:47.743918+0800 TVUAnywhere[9555:2434909] "WebRTCLog":-[TVUWebRTCManager peerConnection:didChangeIceConnectionState:],newState=6,line=859
(RTCLogging.mm:31): (RTCRtpSender.mm:89 -[RTCRtpSender initWithNativeRtpSender:]): RTCRtpSender(0x17400c5c0): created sender: RTCRtpSender {
  senderId: ARDAMSa0112233
}
(webrtcvoiceengine.cc:2352): SetOutputVolume() to 0 for recv stream with ssrc 176080998
(channel.cc:882): Channel disabled
(webrtcvoiceengine.cc:1425): Stopping playout for channel #1
(audio_device_impl.cc:1406): StopPlayout
(audio_device_ios.mm:228): AudioDeviceIOS::StopPlayout
(audio_device_impl.cc:1409): output: 0
(bitrate_allocator.cc:171): UpdateAllocationLimits : total_requested_min_bitrate: 0bps, total_requested_padding_bitrate: 0bps
(audio_device_impl.cc:1446): StopRecording
(audio_device_ios.mm:267): AudioDeviceIOS::StopRecording
(audio_device_ios.mm:934): AudioDeviceIOS::ShutdownPlayOrRecord
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:272 Stop): Stopping audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:284 Stop): Stop,use TVUWebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:294 Uninitialize): Unintializing audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:307 Uninitialize): Uninitialize,use TVUWebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:413 DisposeAudioUnit): Disposing audio unit.
```

`audio unit`停止失败。

在此时，复现了此bug(偶尔会出现)，如图所示：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20170829AURemoteIO/IOThread03.png)

此时，我们发现就是在挂断电话停止`audio unit`时切换了SDK，导致了此bug的发生。


### case5

步骤：(RD:10.12.23.232)

1) 进入TVUCC界面，打通1路电话

2) 在tvucc界面挂断电话，不返回主页面

logs:

```
2017-08-30 09:41:39.077309+0800 TVUAnywhere[9575:2438578] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],hang up phone,phone=112233
2017-08-30 09:41:39.077883+0800 TVUAnywhere[9575:2438578] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----lock-----475
2017-08-30 09:41:39.077961+0800 TVUAnywhere[9575:2438578] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----unlock-----477
2017-08-30 09:41:39.078036+0800 TVUAnywhere[9575:2438578] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],remove mediaStream,phone=112233,line=249
(webrtcsession.cc:820): Session:1168910379541524244 Old state:STATE_RECEIVEDPRANSWER New state:STATE_CLOSED
2017-08-30 09:41:39.089862+0800 TVUAnywhere[9575:2438618] "WebRTCLog":-[TVUWebRTCManager peerConnection:didChangeIceConnectionState:],newState=6,line=859
(RTCLogging.mm:31): (RTCRtpSender.mm:89 -[RTCRtpSender initWithNativeRtpSender:]): RTCRtpSender(0x17000fc30): created sender: RTCRtpSender {
  senderId: ARDAMSa0112233
}
(opensslstreamadapter.cc:703): OpenSSLStreamAdapter::OnEvent SE_READ
(opensslstreamadapter.cc:721):  -- onStreamReadable
(opensslstreamadapter.cc:564): OpenSSLStreamAdapter::Read(2048)
(opensslstreamadapter.cc:626):  -- remote side closed
(webrtcvoiceengine.cc:2352): SetOutputVolume() to 0 for recv stream with ssrc 1778054045
(channel.cc:882): Channel disabled
(webrtcvoiceengine.cc:1425): Stopping playout for channel #1
(audio_device_impl.cc:1406): StopPlayout
(audio_device_ios.mm:228): AudioDeviceIOS::StopPlayout
(audio_device_impl.cc:1409): output: 0
(bitrate_allocator.cc:171): UpdateAllocationLimits : total_requested_min_bitrate: 0bps, total_requested_padding_bitrate: 0bps
(audio_device_impl.cc:1446): StopRecording
(audio_device_ios.mm:267): AudioDeviceIOS::StopRecording
(audio_device_ios.mm:934): AudioDeviceIOS::ShutdownPlayOrRecord
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:272 Stop): Stopping audio unit.

(RTCLogging.mm:31): (voice_processing_audio_unit.mm:280 Stop): Stopped audio unit
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:282 Stop): Stop,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:294 Uninitialize): Unintializing audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:303 Uninitialize): Uninitialized audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:305 Uninitialize): Uninitialize,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:413 DisposeAudioUnit): Disposing audio unit.
```

`audio unit`被成功停止。


### case6

步骤：(RD:10.12.23.232)

1) 进入TVUCC界面，2路电话

2）点击TVUCC界面，挂断1路电话

3）点击返回按钮挂断电话并回到主页面

logs:

```
017-08-30 09:27:43.963249+0800 TVUAnywhere[9548:2433263] "WebRTCLog":hang up phone error,phone=112233
2017-08-30 09:27:43.963262+0800 TVUAnywhere[9548:2433263] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],hang up phone,phone=223344
2017-08-30 09:27:43.963303+0800 TVUAnywhere[9548:2433263] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----lock-----475
2017-08-30 09:27:43.963315+0800 TVUAnywhere[9548:2433263] "rtclock":-[TVUWebRTCManager getPeerConnectionUsePhoneNumber:]----unlock-----477
2017-08-30 09:27:43.970748+0800 TVUAnywhere[9548:2433263] "WebRTCLog":-[TVUWebRTCManager hangupPhone:],remove mediaStream,phone=223344,line=249
(port.cc:1161): Jingle:Conn[0x106a40e00:audio:Bd8VQycb:1:0:local:udp:169.254.66.x:63210->Roli+Ji8:1:2122260223:local:udp:169.254.37.x:57077|CRWS|0|0|9115038255631187454|402]: UpdateState(), ms since last received response=2418, ms since last received data=2, rtt=804, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x106a55e00:audio:G++sID65:1:0:prflx:udp:10.12.24.x:55688->BXgtJXHT:1:2122194687:local:udp:10.12.23.x:60478|C-WI|0|0|7961835276047498750|1218]: UpdateState(), ms since last received response=2501, ms since last received data=527561580, rtt=2436, pings_since_last_response=766d4d65524976787a75684a 
(port.cc:1161): Jingle:Conn[0x106a52e00:audio:CAN/NV9n:1:0:local:udp:192.168.1.x:55688->Roli+Ji8:1:2122260223:local:udp:169.254.37.x:57077|C--I|0|0|9114756780654476798|-]: UpdateState(), ms since last received response=527561580, ms since last received data=527561580, rtt=3000, pings_since_last_response=37744476427730307a6a4465 467636514e71593833493734 354f5476666d79774e534162 443951494e6e6b666e6b4630 6d4e6b6d332f6c774d30506e ... 1 more
(port.cc:1161): Jingle:Conn[0x106a52600:audio://bCSTkM:1:0:local:udp:10.216.132.x:50898->Roli+Ji8:1:2122260223:local:udp:169.254.37.x:57077|C--I|0|0|9114475305677766142|-]: UpdateState(), ms since last received response=527561580, ms since last received data=527561580, rtt=3000, pings_since_last_response=7945557a726a6544564d3069 3859565a4f6d6b4f77774a42 6e416850344c30374c304f62 45696b736a6f7866774e6153 366a6f46314a527657667053 
(port.cc:1161): Jingle:Conn[0x106a55600:audio://bCSTkM:1:0:local:udp:10.216.132.x:50898->BXgtJXHT:1:2122194687:local:udp:10.12.23.x:60478|C--I|0|0|9114475305677635070|-]: UpdateState(), ms since last received response=527561580, ms since last received data=527561580, rtt=3000, pings_since_last_response=78756b7a33555a5051306577 6f637237465737754c512b64 643363495559735578314678 536456614348687055366770 6e785130596e396d33645235 
(port.cc:1161): Jingle:Conn[0x1062a2200:audio://bCSTkM:1:0:local:udp:10.216.132.x:50898->4nPLwL1c:1:2122129151:local:udp:192.168.3.x:63098|C--I|0|0|9114475305677503998|-]: UpdateState(), ms since last received response=527561580, ms since last received data=527561580, rtt=3000, pings_since_last_response=54683372653873727a505642 42786370766a7145794f2f6d 385a6245776c567356725579 51714964642f347359373275 745246366a736972462f7a77 
(port.cc:1161): Jingle:Conn[0x106a56600:audio:Bd8VQycb:1:0:local:udp:169.254.66.x:63210->BXgtJXHT:1:2122194687:local:udp:10.12.23.x:60478|C-xW|0|0|9114756780654476799|-]: UpdateState(), ms since last received response=527561580, ms since last received data=527561580, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x106a3ce00:audio:Bd8VQycb:1:0:local:udp:169.254.66.x:63210->4nPLwL1c:1:2122129151:local:udp:192.168.3.x:63098|C-xW|0|0|9114475305677766143|-]: UpdateState(), ms since last received response=527561580, ms since last received data=527561580, rtt=3000, pings_since_last_response=
(port.cc:1161): Jingle:Conn[0x1062a2a00:audio:CAN/NV9n:1:0:local:udp:192.168.1.x:55688->4nPLwL1c:1:2122129151:local:udp:192.168.3.x:63098|C-xW|0|0|9114475305677635071|-]: UpdateState(), ms since last received response=527561580, ms since last received data=527561580, rtt=3000, pings_since_last_response=
(webrtcsession.cc:820): Session:1248486362969604702 Old state:STATE_RECEIVEDPRANSWER New state:STATE_CLOSED
2017-08-30 09:27:43.979202+0800 TVUAnywhere[9548:2433287] "WebRTCLog":-[TVUWebRTCManager peerConnection:didChangeIceConnectionState:],newState=6,line=859
(RTCLogging.mm:31): (RTCRtpSender.mm:89 -[RTCRtpSender initWithNativeRtpSender:]): RTCRtpSender(0x170203ee0): created sender: RTCRtpSender {
  senderId: ARDAMSa0223344
}
(webrtcvoiceengine.cc:2352): SetOutputVolume() to 0 for recv stream with ssrc 2209835450
(channel.cc:882): Channel disabled
(webrtcvoiceengine.cc:1425): Stopping playout for channel #2
(audio_device_impl.cc:1406): StopPlayout
(audio_device_ios.mm:228): AudioDeviceIOS::StopPlayout
(audio_device_impl.cc:1409): output: 0
(stunrequest.cc:232): Sent STUN request 2; resend delay = 200
(bitrate_allocator.cc:171): UpdateAllocationLimits : total_requested_min_bitrate: 0bps, total_requested_padding_bitrate: 0bps
(audio_device_impl.cc:1446): StopRecording
(audio_device_ios.mm:267): AudioDeviceIOS::StopRecording
(audio_device_ios.mm:934): AudioDeviceIOS::ShutdownPlayOrRecord
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:272 Stop): Stopping audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:284 Stop): Stop,use TVUWebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:294 Uninitialize): Unintializing audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:307 Uninitialize): Uninitialize,use TVUWebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:413 DisposeAudioUnit): Disposing audio unit.
```

`audio unit`停止失败。

## 测试结论

根据以上测试用例，我们得出以下结论：

* `audio unit`有时停止失败，失败的原因是sdk切换造成的
* 停止`audio unit`是在所有电话都挂断后才发生的，也就是最后一路电话挂断时，才会触发`audio unit`停止。`[peerConn close]`触发的。
* 应用中有以下执行顺序：`audio unit`停止、`_peerConn=nil`、`_pcFactory=nil`

我的代码中有以下一段代码：

```
-(void)viewDidAppear:(BOOL)animated
{

    [[TVUWebRTCManager shareInstance] tvuSettingWebRTCSDKType:KTypeWebRTCSDK_TVU];
    if (_debugConsoleView == nil) {
        // add debugConsoleView
        [self.view addSubview:self.debugConsoleView];
        [self.debugConsoleView.debugTextView scrollRangeToVisible:NSMakeRange(self.debugConsoleView.debugTextView.text.length, 1)];
        self.debugConsoleView.hidden = YES;
        
        // add debugParamView
        [self.view addSubview:self.debugParamView];
        self.debugParamView.hidden = YES;
    }
}
```

我在`viewDidAppear:`方法中加入了SDK切换方法。这很可能是问题的所在。经过测试我发现，在`[[TVUWebRTCManager shareInstance] tvuSettingWebRTCSDKType:KTypeWebRTCSDK_TVU];`前面加上10S的休眠时间，以上crash是经常发生的。

### 解决问题

把`viewDidAppear:`中的sdk切换逻辑去掉。因为当前的SDK切换逻辑是这样的：

* 在主页面只能接听电话，所以在接听电话中设置sdk的类型：
	
	```
	- (void)tvuAcceptCall:(NSString *)phone
{
    
    dispatch_queue_t phoneQueue = [self getSerialQueue:phone];
    dispatch_async(phoneQueue, ^{
        [self checkSwitchWebRTCSDKTimestamp];
        [self tvuSettingWebRTCSDKType:KTypeWebRTCSDK_TVU];
    });

    [self beginCommiuncationWithPhoneNumber:phone isAccept:YES];
}
	```
	
* 在tvucc界面只能打电话，所以在打电话的时候设置SDK的类型：

	```
	- (void)tvuCallRequest:(NSArray *)phoneNumbers
{
    dispatch_queue_t queue = [self getSerialQueue:kSerialQueueTag];
    dispatch_async(queue, ^{
        [self checkSwitchWebRTCSDKTimestamp];
        [self tvuSettingWebRTCSDKType:KTypeWebRTCSDK_Original];
    });
    
    if (phoneNumbers == NULL || phoneNumbers == nil || [phoneNumbers count] <= 0) {
        log4cplus_error("WebRTCLog", "The call reqeust array is null..%s",__func__);
        return;
    }
        
    for (NSString *phone in phoneNumbers) {
        dispatch_queue_t phoneQueue = [self getSerialQueue:phone];
        dispatch_async(phoneQueue, ^{
            [self callPhone:phone];
        });
    }
}
	```
	
修改之后，`audio unit`关闭均正常，log如下：

```
RTCLogging.mm:31): (voice_processing_audio_unit.mm:272 Stop): Stopping audio unit.
call AudioQueuePropertyListenerProc  1 
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:280 Stop): Stopped audio unit
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:282 Stop): Stop,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:294 Uninitialize): Unintializing audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:303 Uninitialize): Uninitialized audio unit.
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:305 Uninitialize): Uninitialize,use original WebRTC...
(RTCLogging.mm:31): (voice_processing_audio_unit.mm:413 DisposeAudioUnit): Disposing audio unit.
```

### crash分析流程

* 搞清楚bug复现的场景
* 根据场景分析和场景相关的逻辑
* 首先分析逻辑有没有问题，其次是代码是否正确。一般都是逻辑有问题
* 如果以上步骤都不能帮助bug的解决，那么可以根据crash的调用堆栈信息，去网上搜索crash的线程是和什么模块什么技术相关联的
* 有可能在分析bug的过程中就能复现bug，所以要一边分析一边测试