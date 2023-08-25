---
layout: post
title: 简单直播实现--利用librtmp推音视频流到rtmp服务
description: 实时音视频采集硬编码推流
category: blog
tag: VideoToolBox,librtmp,encoder,live
---

## 概述 

本篇文章介绍在ios平台如何利用rtmp进行推流，进而实现一个简易直播功能，其内容概要如下： 

* 推流服务器搭建
    * 在centos上搭建推流服务器
    * 在mac上搭建推流服务器
* 推流功能
    * 总体业务流程
    * 集成`librtmp`库到应用中
    * 利用rtmp库推音视频流到rtmp服务
* 参考

实例代码： 

* [音视频推流](https://github.com/mediaios/AVLive_Research)
* 欢迎star&fork

代码结构：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_01.png)


运行截图：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_02.png)

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_03.png)


## 推流服务器搭建

下面介绍了两种最常用的rtmp推流服务搭建方式：

* 服务端搭建nginx用于远程推流服务(在云主机搭建，随时随地可以推流)
* 为了避免外网环境差等因素，在本地mac终端搭建nginx用于推流

### centos服务器搭建rtmp推流服务

#### 安装gcc 

nginx编译依赖gcc环境

```
yum -y install gcc gcc-c++
```

#### 安装pcre pcre-devel

nginx的http模块使用pcre来解析正则表达式

```
yum install -y pcre pcre-devel
```

#### 安装zlib

nginx使用zlib对http包的内容进行gzip

```
yum install -y zlib zlib-devel
```

#### 安装open ssl

OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及 SSL 协议，并提供丰富的应用程序供测试或其它目的使用。nginx 不仅支持 http 协议，还支持 https（即在ssl协议上传输http），所以需要在 Centos 安装 OpenSSL 库。

```
yum install -y openssl openssl-devel
```

#### 下载并解压nginx-rtmp-model

```
#下载rtmp包
wget https://github.com/arut/nginx-rtmp-module/archive/master.zip
#解压下载包（centos中默认没有unzip命令，需要yum下载）
unzip -o master.zip
#修改文件夹名
mv nginx-rtmp-module-master nginx-rtmp-module
```

安装nginx

```
#下载nginx
wget http://nginx.org/download/nginx-1.13.8.tar.gz
#解压nignx 
tar -zxvf nginx-1.13.8.tar.gz
#切换到nginx中
cd nginx-1.13.8
#生成配置文件，将上述下载的文件配置到configure中
./configure --prefix=/usr/local/nginx  --add-module=/home/nginx-rtmp-module  --with-http_ssl_module
#编译程序
make
#安装程序
make install
#查看nginx模块
nginx -V
```

#### 修改配置nginx

```
vi /usr/local/nginx/conf/nginx.conf
```

```
#工作进程
worker_processes  1;
#事件配置
events {
    worker_connections  1024;
}
#RTMP配置
rtmp {  
    server {  
        #监听端口
        listen 1935;  
        #
        application myapp {  
            live on;  
        }  
        #hls配置
        application hls {  
            live on;  
            hls on;  
            hls_path /tmp/hls;  
        }  
    }  
}  

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    gzip  on;

    server {
        listen       80;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }
        #配置hls
        location /hls {  
            types {  
                application/vnd.apple.mpegurl m3u8;  
                video/mp2t ts;  
            }  
            root /tmp;  
            add_header Cache-Control no-cache;  
        }  
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

#### 启动nginx

```
/usr/local/nginx/sbin/nginx
```

#### 推送rtmp流

```
ffmpeg -re -i "/home/123.mp4" -vcodec libx264 -vprofile baseline -acodec aac  -ar 44100 -strict -2 -ac 1 -f flv -s 640x480 -q 10 rtmp://localhost:1935/myapp/test1
```

### mac上搭建rtmp服务器 

#### 安装homebrew 

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```


#### 安装nginx 

```
brew tap denji/nginx

brew install nginx-full --with-rtmp-module
```

安装过程中会提示需要安装xcode command line tool,Mac最新场景下安装Xcode时已经没有Command Line了，需要单独安装。根据提示在使用命令xcode-select --install 

在apple官网下载对应的版本并安装 https://developer.apple.com/download/more/   

我当前的开发环境是： 

* macOS Mojave-10.14.5
* XCode version-10.2.1 

所以下载的版本对应如下图所示：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_04.png)


查看nginx安装信息：

```
brew info nginx-full
```

显示如下： 

```
==> Caveats
Docroot is: /usr/local/var/www

The default port has been set in /usr/local/etc/nginx/nginx.conf to 8080 so that
nginx can run without sudo.

nginx will load all files in /usr/local/etc/nginx/servers/.

- Tips -
Run port 80:
 $ sudo chown root:wheel /usr/local/Cellar/nginx-full/1.17.1/bin/nginx
 $ sudo chmod u+s /usr/local/Cellar/nginx-full/1.17.1/bin/nginx
Reload config:
 $ nginx -s reload
Reopen Logfile:
 $ nginx -s reopen
Stop process:
 $ nginx -s stop
Waiting on exit process
 $ nginx -s quit

To have launchd start denji/nginx/nginx-full now and restart at login:
  brew services start denji/nginx/nginx-full
Or, if you don't want/need a background service you can just run:
  nginx

```

nginx安装位置：

```
/usr/local/Cellar/nginx-full/
```

nginx配置文件位置：

```
/usr/local/etc/nginx/nginx.conf
```

nginx服务器根目录位置：

```
/usr/local/var/www
```

验证是否安装成功，执行命令：

```
nginx
```

然后浏览器中输入http://localhost:8080，出现以下界面，证明安装成功

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_05.png)


#### 配置nginx 

`nginx`的配置文件在`/usr/local/etc/nginx`目录下，选择编辑器打开`nginx.conf`文件，在http节点后面添加rtmp配置。

```
http{
    ...
}

#在http节点后面加上rtmp配置
rtmp {
    server {
      # 监听端口
      listen 1935;
      # 分块大小
      chunk_size 4000;

      # RTMP 直播流配置
      application rtmplive {
        # 开启直播的模式
        live on;
        # 设置最大连接数
        max_connections 1024;
      }

      # hls 直播流配置
      application hls {
        live on;
        hls on;
        # 分割文件的存储位置
        hls_path /usr/local/var/www/hls;
        # hls分片大小
        hls_fragment 5s;
      }

    }
}

```

在http节点内的server节点内增加配置： 

```
http {
    ...
    server {
        ...
        location / {
            root   html;
            index  index.html index.htm;
        }

        location /hls {
          # 响应类型
            types {
              application/vnd.apple.mpegurl m3u8;
              video/mp2t ts;
            }
            root /usr/local/var/www;
            # 不要缓存
            add_header Cache-Control no-cache;
        }
        ...
    }
    ...
}
```

配置完成后，使用下面命令重启nginx: 

```
nginx -s stop   // 关闭nginx 
nginx           // 打开nginx

nginx -s reload // 重启nginx
```

#### 测试 

1. 测试rtmp推流 

首先使用ffmpeg网rtmp服务器推流： 

```
ffmpeg -re -i test.mp4 -vcodec libx264 -acodec aac -f flv rtmp://localhost:1935/rtmplive/room1
```

利用ffplay或者vlc查看： 

```
ffplay -i rtmp://localhost:1935/rtmplive/room1
```

2. 测试hls推流 

```
ffmpeg -re -i Test.MOV -vcodec libx264 -acodec aac -f flv rtmp://localhost:1935/hls/stream
```

利用ffplay或者vlc查看：

```
ffplay -i rtmp://localhost:1935/hls/stream
```

也可以在浏览器中输入`http://localhost:8080/hls/stream.m3u8`地址查看hls流： 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_06.png)


## 推流功能

### 流程图 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_07.png)

### 集成librtmp库到应用中

曾经尝试编译出适用于ios平台的`librtmp`库，都由于种种原因没有成功，后续会继续尝试。此处我使用的`librtmp`库是在网上找的一个版本，可以通过以下方式下载：

链接:https://pan.baidu.com/s/1ATEqt31WyNHr8brmEbCLQw  密码:v63u

解压后把库和头文件放到工程中： 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_08.png)

然后正确设置库和头文件的搜索路径。 


### 推流实现

#### 向rtmp服务发送音视频的metadata 

demo中所采集音视频的信息如下： 

* video
	* width: 480
	* height: 640
	* fps: 30
* audio
	* samplerate: 44100
	* BitsPerChannel: 16
	* channels : 1

与rtmp服务器建立连接之后首先要发送音视频的metadata信息。具体代码如下： 

```
- (void)sendMetaData {
    RTMPPacket packet;
    
    char pbuf[2048], *pend = pbuf+sizeof(pbuf);
    
    packet.m_nChannel = 0x03;     // control channel (invoke)
    packet.m_headerType = RTMP_PACKET_SIZE_LARGE;
    packet.m_packetType = RTMP_PACKET_TYPE_INFO;
    packet.m_nTimeStamp = 0;
    packet.m_nInfoField2 = self->rtmp->m_stream_id;
    packet.m_hasAbsTimestamp = TRUE;
    packet.m_body = pbuf + RTMP_MAX_HEADER_SIZE;
    
    char *enc = packet.m_body;
    enc = AMF_EncodeString(enc, pend, &av_setDataFrame);
    enc = AMF_EncodeString(enc, pend, &av_onMetaData);
    
    *enc++ = AMF_OBJECT;
    
    enc = AMF_EncodeNamedNumber(enc, pend, &av_duration,        0.0);
    enc = AMF_EncodeNamedNumber(enc, pend, &av_fileSize,        0.0);
    
    // videosize
    enc = AMF_EncodeNamedNumber(enc, pend, &av_width,           480);
    enc = AMF_EncodeNamedNumber(enc, pend, &av_height,          640);
    
    // video
    enc = AMF_EncodeNamedString(enc, pend, &av_videocodecid,    &av_avc1);
    //640x480
    enc = AMF_EncodeNamedNumber(enc, pend, &av_videodatarate,   480 * 640  / 1000.f);
    enc = AMF_EncodeNamedNumber(enc, pend, &av_framerate,       20);
    
    // audio
    enc = AMF_EncodeNamedString(enc, pend, &av_audiocodecid,    &av_mp4a);
    enc = AMF_EncodeNamedNumber(enc, pend, &av_audiodatarate,   96000);
    
    enc = AMF_EncodeNamedNumber(enc, pend, &av_audiosamplerate, 44100);
    enc = AMF_EncodeNamedNumber(enc, pend, &av_audiosamplesize, 16.0);
    enc = AMF_EncodeNamedBoolean(enc, pend, &av_stereo,     NO);
    
    // sdk version
    enc = AMF_EncodeNamedString(enc, pend, &av_encoder,         &av_SDKVersion);
    
    *enc++ = 0;
    *enc++ = 0;
    *enc++ = AMF_OBJECT_END;
    
    packet.m_nBodySize = enc - packet.m_body;
    if(!RTMP_SendPacket(self->rtmp, &packet, FALSE)) {
        return;
    }
}
```

#### 发送视频sps,pps信息 

sps和pps是需要在其他NALU之前打包推送给服务器。由于RTMP推送的音视频流的封装形式和FLV格式相似，向FMS等流媒体服务器推送H264和AAC直播流时，需要首先发送"AVC sequence header"和"AAC sequence header"（这两项数据包含的是重要的编码信息，没有它们，解码器将无法解码），因此这里的"AVC sequence header"就是用来打包sps和pps的。

AVC sequence header其实就是AVCDecoderConfigurationRecord结构，该结构在标准文档“ISO/IEC-14496-15:2004”的5.2.4.1章节中有详细说明。

如下代码在网上很多地方都能找到，但是却很少有对其每个字节表示什么意思做详细解释。我通过查阅官方文档对其做了详细注释。下面是相关的包结构资料： 

VIDEODATA: 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_09.png)

AVCDecoderConfigurationRecord的定义：

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_10.png)


详细注释具体代码如下： 

```
- (void)sendVideoSps:(NSData *)spsData pps:(NSData *)ppsData
{
    unsigned char* sps = (unsigned char*)spsData.bytes;
    unsigned char* pps = (unsigned char*)ppsData.bytes;
    long sps_len = spsData.length;
    long pps_len = ppsData.length;
    dispatch_async(self.rtmpQueue, ^{
        if(self->rtmp!= NULL)
        {
            unsigned char *body = NULL;
            NSInteger iIndex = 0;
            NSInteger rtmpLength = 1024;
            
            body = (unsigned char *)malloc(rtmpLength);
            memset(body, 0, rtmpLength);
            
            /*** VideoTagHeader: 编码格式为AVC时，该header长度为5 ***/
            body[iIndex++] = 0x17;   // 表示帧类型和CodecID,各占4个bit加一起是1个Byte   1: 表示帧类型，当前是I帧(for AVC, A seekable frame)  7: AVC  元数据当做I帧发送
            body[iIndex++] = 0x00;   // AVCPacketType: 0 = AVC sequence header, 长度为1
            
            body[iIndex++] = 0x00;   // CompositionTime: 0  ,长度为3
            body[iIndex++] = 0x00;
            body[iIndex++] = 0x00;
            
            /*** AVCDecoderConfigurationRecord:包含着H.264解码相关比较重要的sps,pps信息，在给AVC解码器送数据流之前一定要把sps和pps信息先发送，否则解码器不能正常work，而且在
             解码器stop之后再次start之前，如seek，快进快退状态切换等都需要重新发送一遍sps和pps信息。AVCDecoderConfigurationRecord在FLV文件中一般情况也是出现1次，也就是第一个
             video tag.
             ***/
            body[iIndex++] = 0x01;        // 版本 = 1
            body[iIndex++] = sps[1];      // AVCProfileIndication,1个字节长度:
            body[iIndex++] = sps[2];      // profile_compatibility,1个字节长度
            body[iIndex++] = sps[3];      // AVCLevelIndication , 1个字节长度
            body[iIndex++] = 0xff;
            
            // sps
            body[iIndex++] = 0xe1;    // 它的后5位表示SPS数目， 0xe1 = 1110 0001 后五位为 00001 = 1，表示只有1个SPS
            body[iIndex++] = (sps_len >> 8) & 0xff;  // 表示SPS长度：2个字节 ，其存储的就是sps_len (策略：sps长度右移8位&0xff,然后sps长度&0xff)
            body[iIndex++] = sps_len & 0xff;
            memcpy(&body[iIndex], sps, sps_len);
            iIndex += sps_len;
            
            // pps
            body[iIndex++] = 0x01;   // 表示pps的数目，当前表示只有1个pps
            body[iIndex++] = (pps_len >> 8) & 0xff;  // 和sps同理，表示pps的长度：占2个字节 ...
            body[iIndex++] = (pps_len) & 0xff;
            memcpy(&body[iIndex], pps, pps_len);
            iIndex += pps_len;
            
            [self sendPacket:RTMP_PACKET_TYPE_VIDEO data:body size:iIndex nTimestamp:0];
            free(body);
        }
    });
}

```

根据官方文档，对上面的代码关键点解释如下(详细解释在上面代码中)：

* `body[iIndex++] = 0x17;`
	*  表示帧类型和CodecID,各占4个bit加一起是1个Byte  
	*  1: 表示帧类型，当前是I帧(for AVC, A seekable frame)  
	*  7: AVC  元数据当做I帧发送
* `body[iIndex++] = 0xe1; `
	*  它的后5位表示SPS数目， 0xe1 = 1110 0001 后五位为 00001 = 1，表示只有1个SPS

#### 发送视频数据

首先看一下视频数据包的具体结构： 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_11.png)

具体代码实现： 

```
- (void)sendVideoData:(NSData *)data isKeyFrame:(BOOL)isKeyFrame
{
    __block uint32_t length = data.length;
    dispatch_async(self.rtmpQueue, ^{
        if(self->rtmp != NULL)
        {
            uint32_t timeoffset = [[NSDate date] timeIntervalSince1970]*1000 - self->start_time;  /*start_time为开始直播时的时间戳*/
            NSInteger i = 0;
            NSInteger rtmpLength = data.length + 9;
            unsigned char *body = (unsigned char *)malloc(rtmpLength);
            memset(body, 0, rtmpLength);
            
            if (isKeyFrame) {
                body[i++] = 0x17;        // 1:Iframe  7:AVC
            } else {
                body[i++] = 0x27;        // 2:Pframe  7:AVC
            }
            body[i++] = 0x01;    // AVCPacketType:   0 表示AVC sequence header; 1 表示AVC NALU; 2 表示AVC end of sequence....
            body[i++] = 0x00;    // CompositionTime，占3个字节: 1表示 Composition time offset; 其它情况都是0
            body[i++] = 0x00;
            body[i++] = 0x00;
            body[i++] = (data.length >> 24) & 0xff;  // NALU size
            body[i++] = (data.length >> 16) & 0xff;
            body[i++] = (data.length >>  8) & 0xff;
            body[i++] = (data.length) & 0xff;
            memcpy(&body[i], data.bytes, data.length);  // NALU data
            
            [self sendPacket:RTMP_PACKET_TYPE_VIDEO data:body size:(rtmpLength) nTimestamp:timeoffset];
            free(body);
        }
    });
}
```


#### 发送音频header

音频header主要是音频的一些信息，此时包的第二个字节要为0. 

![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_12.png)


![](https://raw.githubusercontent.com/mediaios/AVLive_Research/master/imgs/20190703-rtmp_13.png)


具体代码： 

```
- (void)sendAudioHeader:(NSData *)data{
    
    NSInteger audioLength = data.length;
    dispatch_async(self.rtmpQueue, ^{
        NSInteger rtmpLength = audioLength + 2;     /*spec data长度,一般是2*/
        unsigned char *body = (unsigned char *)malloc(rtmpLength);
        memset(body, 0, rtmpLength);
        
        /*AF 00 + AAC RAW data*/
        body[0] = 0xAE;    // 4bit表示音频格式， 10表示AAC，所以用A来表示。  A: 表示发送的是AAC ； SountRate占2bit,此处是44100用3表示，转化为二进制位 11 ； SoundSize占1个bit,0表示8位，1表示16位，此处是16位用1表示，二进制表示为 1； SoundType占1个bit,0表示单声道，1表示立体声，此处是单声道用0表示，二进制表示为 0； 1110 = E
        body[1] = 0x00;  // 0表示的是audio的配置
        memcpy(&body[2], data.bytes, audioLength);          /*spec_buf是AAC sequence header数据*/
        [self sendPacket:RTMP_PACKET_TYPE_AUDIO data:body size:rtmpLength nTimestamp:0];
        free(body);
    });
}
```

根据官方文档，对上述代码做解释： 

* `body[0] = 0xAE`：

	* 该字节分为4部分，前4bit表示音频格式，10表示AAC,因此用A来表示
	* 接下来的2个bit表示采样率SountRate，此处为44100，用3表示，转化为二进制为 11 
	* 接下来的1个bit表示SoundSize,此处为16，用1表示，转化为二进制位 1
	* 接下来的1个bit表示SoundType,0表示单声道，1表示立体声，此处是单声道用0表示，二进制表示为 0；
	*  综上，后面4个bit表示为 1110=>0xE  所以最后表示为 0xAE

* `body[1] = 0x00`:
	*  0表示的是audio的配置


#### 发送音频数据

和音频的头相比，变化的仅仅是第二个字节。

```
- (void)sendAudioData:(NSData *)data{
    NSInteger audioLength = data.length;
    dispatch_async(self.rtmpQueue, ^{
        uint32_t timeoffset = [[NSDate date] timeIntervalSince1970]*1000 - self->start_time;
        NSInteger rtmpLength = audioLength + 2;    /*spec data长度,一般是2*/
        unsigned char *body = (unsigned char *)malloc(rtmpLength);
        memset(body, 0, rtmpLength);
        
        /*AF 01 + AAC RAW data*/
        body[0] = 0xAE;
        body[1] = 0x01;
        memcpy(&body[2], data.bytes, audioLength);
        [self sendPacket:RTMP_PACKET_TYPE_AUDIO data:body size:rtmpLength nTimestamp:timeoffset];
        free(body);
    });
}
```

## 参考

* [Video File Format Specification Version 10](https://www.adobe.com/content/dam/acom/en/devnet/flv/video_file_format_spec_v10.pdf)
* [INTERNATIONAL STANDARD ISO/IEC 14496-16](https://www.sis.se/api/document/preview/910072/)
* [H.264/AVC编码的FLV文件的第二个Tag: AVCDecoderConfigurationRecord](https://www.jianshu.com/p/e1e417eee2e7)
* [FLV视频封装格式详解](http://www.360doc.com/content/12/1212/15/4550476_253606537.shtml)
* [RTMP协议分析及H.264打包原理](http://www.voidcn.com/article/p-ffsjepvk-hh.html)