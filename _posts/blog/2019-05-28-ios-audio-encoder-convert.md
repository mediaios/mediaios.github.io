---
layout: post
title: 利用AudioQueue做音频采集编码和播放
description: 利用AudioQueue做音频采集编码和播放
category: blog
tag: AudioQueue,pcm,aac,audio convert
---

## 概述

在直播应用开发中我们经常需要实时对音频做处理，比如音频录制、播放、编码等。本文介绍的是使用`AudioQueue`对音频做上述处理。

PCM和AAC是音频的两种不同的格式，PCM是无损音频数据，AAC是压缩编码过的数据。我们在介绍`AudioQueue`的用法之前，首先对音频的这两种格式做大致了解。关于音频的基础请参考 [音频基础知识](https://juejin.im/post/5ced12e6f265da1b5d578bb5)

文章目录：

* AAC音频
* AudioQueue录制音频原始帧PCM数据
* AudioQueue播放PCM音频文件 
* AudioQueue录制音频PCM数据，同时转化为aac数据保存到沙盒中，最后利用`ffmpeg`命令查看并播放aac文件 

示例代码： 

* [AudioQueue录制音频原始帧PCM数据](https://github.com/mediaios/AVLive_Research)
* [AudioQueue播放PCM音频文件](https://github.com/mediaios/AVLive_Research)
* [AudioQueue录制PCM同时把PCM编码成AAC](https://github.com/mediaios/AVLive_Research)
* 欢迎star&fork

代码结构: 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_01.png)

## AAC音频 

因为我们会介绍从PCM转化为AAC，所以我们先对AAC有一个大致的了解。

AAC的音频文件格式有两种分别是ADIF 和 ADTS 。

### ADIF和ADTS
 
 * ADIF : (Audio Data Interchange Format )音频数据交换格式。这种格式的特征是可以确定的找到这个音频数据的开始，不需进行在音频数据流中间开始的解码，即它的解码必须在明确定义的开始处进行。故这种格式常用在磁盘文件中。
 * ADTS : (Audio Data Transport Stream)音频数据传输流。这种格式的特征是它是一个有同步字的比特流，解码可以在这个流中任何位置开始。它的特征类似于mp3数据流格式。

 简单说，ADTS可以在任意帧解码，也就是说它每一帧都有头信息。ADIF只有一个统一的头，所以必须得到所有的数据后解码。且这两种的header的格式也是不同的，目前一般编码后的和抽取出的都是ADTS格式的音频流。
 
 ADIF格式:
 
![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_02.png)
 
 ADTS格式:
 	
![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_03.png)
 
### ADIF和ADTS的头信息

ADIF的头信息如图：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_04.png)

ADIF头信息位于AAC文件的起始处，接下来就是连续的 raw data blocks。组成 ADIF头信息的各个域如下所示：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_05.png)

ADTS的固定头信息：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_06.png)

ADTS的可变头信息：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_07.png)

* 帧同步目的在于找出帧头在比特流中的位置，13818-7规定，aac ADTS格式的帧头。同步字为12比特的“1111 1111 1111”
* ADTS的头信息为两部分组成，其一为固定头信息，紧接着是可变头信息。固定头信息中的数据每一帧都相同，而可变头信息则在帧与帧之间可变

### AAC元素信息

 在AAC中，原始数据块的组成可能有六种不同的元素：
 
 * SCE: Single Channel Element单通道元素。单通道元素基本上只由一个ICS组成。一个原始数据块最可能由16个SCE组成。
 * CPE: Channel Pair Element 双通道元素，由两个可能共享边信息的ICS和一些联合立体声编码信息组成。一个原始数据块最多可能由16个SCE组成。
 *  CCE: Coupling Channel Element 藕合通道元素。代表一个块的多通道联合立体声信息或者多语种程序的对话信息。
 *  LFE: Low Frequency Element 低频元素。包含了一个加强低采样频率的通道。
 *  DSE: Data Stream Element 数据流元素，包含了一些并不属于音频的附加信息。
 *  PCE: Program Config Element 程序配置元素。包含了声道的配置信息。它可能出现在ADIF 头部信息中。
 *  FIL: Fill Element 填充元素。包含了一些扩展信息。如SBR，动态范围控制信息等。
 
## AudioQueue录制PCM音频 

关于什么是`AudioQueue`及其介绍在此不再描述，请查阅苹果官方文档或[译:Audio Queue Services Programming Guide](https://juejin.im/post/5cdb8a88518825123570f4f3)

### 设置录制音频格式

音频格式：PCM ; 采样率：48K ; 每帧数据通道数：1 ; 一帧数据中每个通道的样本数据位数: 16 ; 下面是具体代码： 

```
- (void)settingAudioFormat
{
    /*** setup audio sample rate , channels number, and format ID ***/
    memset(&dataFormat, 0, sizeof(dataFormat));
    UInt32 size = sizeof(dataFormat.mSampleRate);
    AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareSampleRate, &size, &dataFormat.mSampleRate);
    dataFormat.mSampleRate = kAudioSampleRate;
    size = sizeof(dataFormat.mChannelsPerFrame);
    AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareInputNumberChannels, &size, &dataFormat.mChannelsPerFrame);
    dataFormat.mFormatID = kAudioFormatLinearPCM;
    dataFormat.mChannelsPerFrame = 1;
    dataFormat.mFormatFlags = kLinearPCMFormatFlagIsSignedInteger | kLinearPCMFormatFlagIsPacked;
    dataFormat.mBitsPerChannel = 16;
    dataFormat.mBytesPerPacket = dataFormat.mBytesPerFrame = (dataFormat.mBitsPerChannel / 8) * dataFormat.mChannelsPerFrame;
    dataFormat.mFramesPerPacket = kAudioFramesPerPacket; // AudioQueue collection pcm data , need to set as this
}
```

### 创建AudioQueue 

```
extern OSStatus             
AudioQueueNewInput(                 const AudioStreamBasicDescription *inFormat,
                                    AudioQueueInputCallback         inCallbackProc,
                                    void * __nullable               inUserData,
                                    CFRunLoopRef __nullable         inCallbackRunLoop,
                                    CFStringRef __nullable          inCallbackRunLoopMode,
                                    UInt32                          inFlags,
                                    AudioQueueRef __nullable * __nonnull outAQ);
```
* inFormt: 所录制音频的格式，是`AudioStreamBasicDescription`的实例。`AudioStreamBasicDescription`是对音频格式的描述。
* inCallbackProc : 是一个回调，当一个buffer被填充完成时，会触发这个回调。
* inCallbackRunLoop：要调用inCallbackProc的事件循环。如果指定NULL，则在其中一个音频队列的内部线程上调用回调。这个参数一般填写NULL
* inCallbackRunLoopMode：为RunLoop模式，如果传入NULL就相当于kCFRunLoopCommonModes，一般这个参数也是填写NULL 
* inFlags : 保留字段，直接传0 
* outAQ: 返回生成的AudioQueue实例,返回值用来判断是否成功创建（OSStatus == noErr)

下面代码时创建录制的AudioQueue的代码： 

```
- (void)settingCallBackFunc
{
    /*** 设置录音回调函数 ***/
    OSStatus status = 0;
    //    int bufferByteSize = 0;
    UInt32 size = sizeof(dataFormat);
    status = AudioQueueNewInput(&dataFormat, inputAudioQueueBufferHandler, (__bridge void *)self, NULL, NULL, 0, &mQueue);
    if (status != noErr) {
        NSLog(@"AppRecordAudio,%s,AudioQueueNewInput failed status:%d ",__func__,(int)status);
    }
    
    for (int i = 0 ; i < kQueueBuffers; i++) {
        status = AudioQueueAllocateBuffer(mQueue, kAudioPCMTotalPacket * kAudioBytesPerPacket * dataFormat.mChannelsPerFrame, &mBuffers[i]);
        status = AudioQueueEnqueueBuffer(mQueue, mBuffers[i], 0, NULL);
    }
}
```

### 音频录制回调函数 

在此回调中，我们直接把PCM数据写到沙盒中(只写前800帧，数目达到时会停止录制)。

```
/*!
 @discussion
 AudioQueue 音频录制回调函数
 @param      inAQ
 回调函数的音频队列.
 @param      inBuffer
 是一个被音频队列填充新的音频数据的音频队列缓冲区，它包含了回调函数写入文件所需要的新数据.
 @param      inStartTime
 是缓冲区中的一采样的参考时间
 @param      inNumberPacketDescriptions
 参数中包描述符（packet descriptions）的数量，如果你正在录制一个VBR(可变比特率（variable bitrate））格式, 音频队列将会提供这个参数给你的回调函数，这个参数可以让你传递给AudioFileWritePackets函数. CBR (常量比特率（constant bitrate）) 格式不使用包描述符。对于CBR录制，音频队列会设置这个参数并且将inPacketDescs这个参数设置为NULL
 
 */
static void inputAudioQueueBufferHandler(void * __nullable               inUserData,
                               AudioQueueRef                   inAQ,
                               AudioQueueBufferRef             inBuffer,
                               const AudioTimeStamp *          inStartTime,
                               UInt32                          inNumberPacketDescriptions,
                               const AudioStreamPacketDescription * __nullable inPacketDescs)
{
    if (!inUserData) {
        NSLog(@"AppRecordAudio,%s,inUserData is null",__func__);
        return;
    }
    
    NSLog(@"%s, audio length: %d",__func__,inBuffer->mAudioDataByteSize);
    static int createCount = 0;
    static FILE *fp_pcm = NULL;
    if (createCount == 0) {
        NSString *paths = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];
        NSString *debugUrl = [paths stringByAppendingPathComponent:@"debug"] ;
        NSFileManager *fileManager = [NSFileManager defaultManager];
        [fileManager createDirectoryAtPath:debugUrl withIntermediateDirectories:YES attributes:nil error:nil];
        
        NSString *audioFile = [paths stringByAppendingPathComponent:@"debug/queue_pcm_48k.pcm"] ;
        fp_pcm = fopen([audioFile UTF8String], "wb++");
    }
    createCount++;
    
    MIAudioQueueRecord *miAQ = (__bridge MIAudioQueueRecord *)inUserData;
    if (createCount <= 800) {
        void *bufferData = inBuffer->mAudioData;
        UInt32 buffersize = inBuffer->mAudioDataByteSize;
        fwrite((uint8_t *)bufferData, 1, buffersize, fp_pcm);
    }else{
        fclose(fp_pcm);
        NSLog(@"AudioQueue, close PCM file ");
        [miAQ stopRecorder];
        createCount = 0;
    }
    
    if (miAQ.m_isRunning) {
        AudioQueueEnqueueBuffer(inAQ, inBuffer, 0, NULL);
    }
}
```

### 开始录制和停止录制 

开始录制： 

```
- (void)startRecorder
{
    [self createAudioSession];
    [self settingAudioFormat];
    [self settingCallBackFunc];
    
    if (self.m_isRunning) {
        return;
    }

    /*** start audioQueue ***/
    OSStatus status = AudioQueueStart(mQueue, NULL);
    if (status != noErr) {
        NSLog(@"AppRecordAudio,%s,AudioQueueStart failed status:%d  ",__func__,(int)status);
    }
    self.m_isRunning = YES;
}
```

停止录制： 

```
- (void)stopRecorder
{
    if (!self.m_isRunning) {
        return;
    }
    self.m_isRunning = NO;
    
    if (mQueue) {
        OSStatus stopRes = AudioQueueStop(mQueue, true);
        
        if (stopRes == noErr) {
            for (int i = 0; i < kQueueBuffers; i++) {
                AudioQueueFreeBuffer(mQueue, mBuffers[i]);
            }
        }else{
            NSLog(@"AppRecordAudio,%s,stop AudioQueue failed.  ",__func__);
        }
        
        AudioQueueDispose(mQueue, true);
        mQueue = NULL;
    }
}
```

### 利用ffplay播放录制的音频

我们此时进入到沙河目录中先用ffplay命令播放一下录制好的pcm音频，后面我们会利用AudioQueue来播放pcm文件。 

```
ffplay -f s16le -ar 48000 -ac 1 queue_pcm_48k.pcm 
```

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_08.png)


## AudioQueue播放PCM文件 

我们在本小结中，就播放上面刚刚录制的pcm文件。 

### 设置待播放的音频格式

```
- (void)settingAudioFormat
{
    /*** setup audio sample rate , channels number, and format ID ***/
    memset(&dataFormat, 0, sizeof(dataFormat));
    UInt32 size = sizeof(dataFormat.mSampleRate);
    AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareSampleRate, &size, &dataFormat.mSampleRate);
    dataFormat.mSampleRate = kAudioSampleRate;
    size = sizeof(dataFormat.mChannelsPerFrame);
    AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareInputNumberChannels, &size, &dataFormat.mChannelsPerFrame);
    dataFormat.mFormatID = kAudioFormatLinearPCM;
    dataFormat.mChannelsPerFrame = 1;
    dataFormat.mFormatFlags = kLinearPCMFormatFlagIsSignedInteger | kLinearPCMFormatFlagIsPacked;
    dataFormat.mBitsPerChannel = 16;
    dataFormat.mBytesPerPacket = dataFormat.mBytesPerFrame = (dataFormat.mBitsPerChannel / 8) * dataFormat.mChannelsPerFrame;
    dataFormat.mFramesPerPacket = kAudioFramesPerPacket; // AudioQueue collection pcm data , need to set as this
}
```

### 创建播放的AudioQueue并设置callback 

```
- (void)settingCallBackFunc
{
    /*** 设置录音回调函数 ***/
    OSStatus status = 0;
    //    int bufferByteSize = 0;
    UInt32 size = sizeof(dataFormat);
    
    /*** 设置播放回调函数 ***/
    status = AudioQueueNewOutput(&dataFormat,
                                 miAudioPlayCallBack,
                                 (__bridge void *)self,
                                 NULL,
                                 NULL,
                                 0,
                                 &mQueue);
    if (status != noErr) {
        NSLog(@"AppRecordAudio,%s, AudioQueueNewOutput failed status:%d",__func__,(int)status);
    }
    
    for (int i = 0 ; i < kQueueBuffers; i++) {
        status = AudioQueueAllocateBuffer(mQueue, kAudioPCMTotalPacket * kAudioBytesPerPacket * dataFormat.mChannelsPerFrame, &mBuffers[i]);
        status = AudioQueueEnqueueBuffer(mQueue, mBuffers[i], 0, NULL);
    }

}

```

### 从pcm文件中读音频数据 

```
- (void)initPlayedFile
{
    NSString *paths = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];
    NSString *audioFile = [paths stringByAppendingPathComponent:@"debug/queue_pcm_48k.pcm"] ;
    NSFileManager *manager = [NSFileManager defaultManager];
    NSLog(@"file exist = %d",[manager fileExistsAtPath:audioFile]);
    NSLog(@"file size = %lld",[[manager attributesOfItemAtPath:audioFile error:nil] fileSize]) ;
    file  = fopen([audioFile UTF8String], "r");
    if(file)
    {
        fseek(file, 0, SEEK_SET);
        pcmDataBuffer = malloc(1024);
    }
    else{
        NSLog(@"!!!!!!!!!!!!!!!!");
    }
    synlock = [[NSLock alloc] init];
}

```

### 把pcm数据送给AudioQueue Buffer播放

```
- (void)startPlay
{
    [self initPlayedFile];
    [self createAudioSession];
    [self settingAudioFormat];
    [self settingCallBackFunc];
    
    AudioQueueStart(mQueue, NULL);
    for (int i = 0; i < kQueueBuffers; i++) {
        [self readPCMAndPlay:mQueue buffer:mBuffers[i]];
    }
}

-(void)readPCMAndPlay:(AudioQueueRef)outQ buffer:(AudioQueueBufferRef)outQB
{
    [synlock lock];
    int readLength = fread(pcmDataBuffer, 1, 1024, file);//读取文件
    NSLog(@"read raw data size = %d",readLength);
    outQB->mAudioDataByteSize = readLength;
    Byte *audiodata = (Byte *)outQB->mAudioData;
    for(int i=0;i<readLength;i++)
    {
        audiodata[i] = pcmDataBuffer[i];
    }
    /*
     将创建的buffer区添加到audioqueue里播放
     AudioQueueBufferRef用来缓存待播放的数据区，AudioQueueBufferRef有两个比较重要的参数，AudioQueueBufferRef->mAudioDataByteSize用来指示数据区大小，AudioQueueBufferRef->mAudioData用来保存数据区
     */
    AudioQueueEnqueueBuffer(outQ, outQB, 0, NULL);
    [synlock unlock];
}
```

### 运行 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_09.png)

如上图所是，点击播放会播放上一次录制的pcm音频。 


## AudioQueue实时编码PCM数据为AAC并保存到沙盒中 

在本小结中，我们首先启动`AudioQueue`录制pcm音频，同时创建一个转化器把PCM数据转化成AAC，最后把AAC保存到沙盒中。流程大致如下： 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_10.png)

采样率等参数依然和前面录制和播放的保持一致，都是48K 

### 设置输入输出音频编码参数并创建转化器

```
- (void)settingInputAudioFormat
{
    /*** setup audio sample rate , channels number, and format ID ***/
    memset(&inAudioStreamDes, 0, sizeof(inAudioStreamDes));
    UInt32 size = sizeof(inAudioStreamDes.mSampleRate);
    AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareSampleRate, &size, &inAudioStreamDes.mSampleRate);
    inAudioStreamDes.mSampleRate = kAudioSampleRate;
    size = sizeof(inAudioStreamDes.mChannelsPerFrame);
    AudioSessionGetProperty(kAudioSessionProperty_CurrentHardwareInputNumberChannels, &size, &inAudioStreamDes.mChannelsPerFrame);
    inAudioStreamDes.mFormatID = kAudioFormatLinearPCM;
    inAudioStreamDes.mChannelsPerFrame = 1;
    inAudioStreamDes.mFormatFlags = kLinearPCMFormatFlagIsSignedInteger | kLinearPCMFormatFlagIsPacked;
    inAudioStreamDes.mBitsPerChannel = 16;
    inAudioStreamDes.mBytesPerPacket = inAudioStreamDes.mBytesPerFrame = (inAudioStreamDes.mBitsPerChannel / 8) * inAudioStreamDes.mChannelsPerFrame;
    inAudioStreamDes.mFramesPerPacket = kAudioFramesPerPacket; // AudioQueue collection pcm data , need to set as this
}

- (void)settingDestAudioStreamDescription
{
    outAudioStreamDes.mSampleRate = kAudioSampleRate;
    outAudioStreamDes.mFormatID = kAudioFormatMPEG4AAC;
    outAudioStreamDes.mBytesPerPacket = 0;
    outAudioStreamDes.mFramesPerPacket = 1024;
    outAudioStreamDes.mBytesPerFrame = 0;
    outAudioStreamDes.mChannelsPerFrame = 1;
    outAudioStreamDes.mBitsPerChannel = 0;
    outAudioStreamDes.mReserved = 0;
    AudioClassDescription *des = [self getAudioClassDescriptionWithType:kAudioFormatMPEG4AAC
                                                       fromManufacturer:kAppleSoftwareAudioCodecManufacturer];
    OSStatus status = AudioConverterNewSpecific(&inAudioStreamDes, &outAudioStreamDes, 1, des, &miAudioConvert);
    if (status != 0) {
        NSLog(@"create convert failed...\n");
    }
    
    UInt32 targetSize   = sizeof(outAudioStreamDes);
    UInt32 bitRate  =  64000;
    targetSize      = sizeof(bitRate);
    status          = AudioConverterSetProperty(miAudioConvert,
                                                kAudioConverterEncodeBitRate,
                                                targetSize, &bitRate);
    if (status != noErr) {
        NSLog(@"set bitrate error...");
        return;
    }
}
```

### 获取编解码器 

```
/**
 *  获取编解码器
 *  @param type         编码格式
 *  @param manufacturer 软/硬编
 *  @return 指定编码器
 */
- (AudioClassDescription *)getAudioClassDescriptionWithType:(UInt32)type
                                           fromManufacturer:(UInt32)manufacturer
{
    static AudioClassDescription desc;
    
    UInt32 encoderSpecifier = type;
    OSStatus st;
    
    UInt32 size;
    // 取得给定属性的信息
    st = AudioFormatGetPropertyInfo(kAudioFormatProperty_Encoders,
                                    sizeof(encoderSpecifier),
                                    &encoderSpecifier,
                                    &size);
    if (st) {
        NSLog(@"error getting audio format propery info: %d", (int)(st));
        return nil;
    }
    
    unsigned int count = size / sizeof(AudioClassDescription);
    AudioClassDescription descriptions[count];
    // 取得给定属性的数据
    st = AudioFormatGetProperty(kAudioFormatProperty_Encoders,
                                sizeof(encoderSpecifier),
                                &encoderSpecifier,
                                &size,
                                descriptions);
    if (st) {
        NSLog(@"error getting audio format propery: %d", (int)(st));
        return nil;
    }
    
    for (unsigned int i = 0; i < count; i++) {
        if ((type == descriptions[i].mSubType) &&
            (manufacturer == descriptions[i].mManufacturer)) {
            memcpy(&desc, &(descriptions[i]), sizeof(desc));
            return &desc;
        }
    }
    
    return nil;
}
```

### 填充PCM到缓冲区 

```
/**
 *  填充PCM到缓冲区
 */
- (size_t) copyPCMSamplesIntoBuffer:(AudioBufferList*)ioData {
    size_t originalBufferSize = _pcmBufferSize;
    if (!originalBufferSize) {
        return 0;
    }
    ioData->mBuffers[0].mData = _pcmBuffer;
    ioData->mBuffers[0].mDataByteSize = (int)_pcmBufferSize;
    _pcmBuffer = NULL;
    _pcmBufferSize = 0;
    return originalBufferSize;
}
```

### AAC的DTS计算 

具体可参考： 

* [Advanced Audio Coding](https://wiki.multimedia.cx/index.php/Advanced_Audio_Coding)
* [MPEG-4 Audio](https://wiki.multimedia.cx/index.php?title=MPEG-4_Audio)

下面代码时支持采样率为48K，通道数为1的DTS计算： 

```
- (NSData*)adtsDataForPacketLength:(NSUInteger)packetLength {
    int adtsLength = 7;
    char *packet = malloc(sizeof(char) * adtsLength);
    // Variables Recycled by addADTStoPacket
    int profile = 2;  //AAC LC
    //39=MediaCodecInfo.CodecProfileLevel.AACObjectELD;
    int freqIdx = 3;  //48KHz
    int chanCfg = 1;  //MPEG-4 Audio Channel Configuration. 1 Channel front-center
    NSUInteger fullLength = adtsLength + packetLength;
    // fill in ADTS data
    packet[0] = (char)0xFF; // 11111111     = syncword
    packet[1] = (char)0xF9; // 1111 1 00 1  = syncword MPEG-2 Layer CRC
    packet[2] = (char)(((profile-1)<<6) + (freqIdx<<2) +(chanCfg>>2));
    packet[3] = (char)(((chanCfg&3)<<6) + (fullLength>>11));
    packet[4] = (char)((fullLength&0x7FF) >> 3);
    packet[5] = (char)(((fullLength&7)<<5) + 0x1F);
    packet[6] = (char)0xFC;
    NSData *data = [NSData dataWithBytesNoCopy:packet length:adtsLength freeWhenDone:YES];
    return data;
}
```

### PCM转AAC并把AAC写入到沙盒

```
static int initTime = 0;
- (void)encodePCMToAAC:(MIAudioQueueConvert *)convert
{
    if (initTime == 0) {
        initTime = 1;
        [self settingDestAudioStreamDescription];
    }
    OSStatus status;
    memset(_aacBuffer, 0, _aacBufferSize);
    
    AudioBufferList *bufferList             = (AudioBufferList *)malloc(sizeof(AudioBufferList));
    bufferList->mNumberBuffers              = 1;
    bufferList->mBuffers[0].mNumberChannels = outAudioStreamDes.mChannelsPerFrame;
    bufferList->mBuffers[0].mData           = _aacBuffer;
    bufferList->mBuffers[0].mDataByteSize   = (int)_aacBufferSize;
    
    AudioStreamPacketDescription outputPacketDescriptions;
    UInt32 inNumPackets = 1;
    status = AudioConverterFillComplexBuffer(miAudioConvert,
                                             pcmEncodeConverterInputCallback,
                                             (__bridge void *)(self),//inBuffer->mAudioData,
                                             &inNumPackets,
                                             bufferList,
                                             &outputPacketDescriptions);
    
    if (status == noErr) {
        NSData *aacData = [NSData dataWithBytes:bufferList->mBuffers[0].mData length:bufferList->mBuffers[0].mDataByteSize];
        static int createCount = 0;
        static FILE *fp_aac = NULL;
        if (createCount == 0) {
            NSString *paths = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) objectAtIndex:0];
            NSString *debugUrl = [paths stringByAppendingPathComponent:@"debug"] ;
            NSFileManager *fileManager = [NSFileManager defaultManager];
            [fileManager createDirectoryAtPath:debugUrl withIntermediateDirectories:YES attributes:nil error:nil];
            
            NSString *audioFile = [paths stringByAppendingPathComponent:@"debug/queue_aac_48k.aac"] ;
            fp_aac = fopen([audioFile UTF8String], "wb++");
        }
        createCount++;
        if (createCount <= 800) {
            NSData *rawAAC = [NSData dataWithBytes:bufferList->mBuffers[0].mData length:bufferList->mBuffers[0].mDataByteSize];
            NSData *adtsHeader = [self adtsDataForPacketLength:rawAAC.length];
            NSMutableData *fullData = [NSMutableData dataWithData:adtsHeader];
            [fullData appendData:rawAAC];
            
            void * bufferData = fullData.bytes;
            int buffersize = fullData.length;
            
            fwrite((uint8_t *)bufferData, 1, buffersize, fp_aac);
        }else{
            fclose(fp_aac);
            NSLog(@"AudioQueue, close aac file ");
            [self stopRecorder];
            createCount = 0;
        }
    }
}
```

### 利用ffplay播放 

ffplay命令： 

```
ffplay -ar 48000 queue_aac_48k.aac 
```

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190518_aq_11.png)
