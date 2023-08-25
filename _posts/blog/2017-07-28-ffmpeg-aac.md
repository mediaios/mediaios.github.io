---
layout: post
title: AAC音频码流解析
description: 网上资料系统整理——AAC音频码流解析
category: blog
tag: ffmpeg
---

## 说明

本文中的程序是一个AAC码流解析程序。该程序可以从AAC码流中分析得到它的基本单元ADTS frame，并且可以简单解析ADTS frame首部的字段。通过修改该程序可以实现不同的AAC码流处理功能。

## 原理

AAC原始码流（又称为“裸流”）是由一个一个的ADTS frame组成的。他们的结构如下图所示。

![](http://img.blog.csdn.net/20160118101611729)

其中每个ADTS frame之间通过syncword（同步字）进行分隔。同步字为0xFFF（二进制“111111111111”）。AAC码流解析的步骤就是首先从码流中搜索0x0FFF，分离出ADTS frame；然后再分析ADTS frame的首部各个字段。本文的程序即实现了上述的两个步骤。

## 代码

整个程序位于simplest_aac_parser()函数中，如下所示。

```
/** 
 * 最简单的视音频数据处理示例 
 * Simplest MediaData Test 
 * 
 * 雷霄骅 Lei Xiaohua 
 * leixiaohua1020@126.com 
 * 中国传媒大学/数字电视技术 
 * Communication University of China / Digital TV Technology 
 * http://blog.csdn.net/leixiaohua1020 
 * 
 * 本项目包含如下几种视音频测试示例： 
 *  (1)像素数据处理程序。包含RGB和YUV像素格式处理的函数。 
 *  (2)音频采样数据处理程序。包含PCM音频采样格式处理的函数。 
 *  (3)H.264码流分析程序。可以分离并解析NALU。 
 *  (4)AAC码流分析程序。可以分离并解析ADTS帧。 
 *  (5)FLV封装格式分析程序。可以将FLV中的MP3音频码流分离出来。 
 *  (6)UDP-RTP协议分析程序。可以将分析UDP/RTP/MPEG-TS数据包。 
 * 
 * This project contains following samples to handling multimedia data: 
 *  (1) Video pixel data handling program. It contains several examples to handle RGB and YUV data. 
 *  (2) Audio sample data handling program. It contains several examples to handle PCM data. 
 *  (3) H.264 stream analysis program. It can parse H.264 bitstream and analysis NALU of stream. 
 *  (4) AAC stream analysis program. It can parse AAC bitstream and analysis ADTS frame of stream. 
 *  (5) FLV format analysis program. It can analysis FLV file and extract MP3 audio stream. 
 *  (6) UDP-RTP protocol analysis program. It can analysis UDP/RTP/MPEG-TS Packet. 
 * 
 */  
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
  
  
int getADTSframe(unsigned char* buffer, int buf_size, unsigned char* data ,int* data_size){  
    int size = 0;  
  
    if(!buffer || !data || !data_size ){  
        return -1;  
    }  
  
    while(1){  
        if(buf_size  < 7 ){  
            return -1;  
        }  
        //Sync words  
        if((buffer[0] == 0xff) && ((buffer[1] & 0xf0) == 0xf0) ){  
            size |= ((buffer[3] & 0x03) <<11);     //high 2 bit  
            size |= buffer[4]<<3;                //middle 8 bit  
            size |= ((buffer[5] & 0xe0)>>5);        //low 3bit  
            break;  
        }  
        --buf_size;  
        ++buffer;  
    }  
  
    if(buf_size < size){  
        return 1;  
    }  
  
    memcpy(data, buffer, size);  
    *data_size = size;  
  
    return 0;  
}  
  
int simplest_aac_parser(char *url)  
{  
    int data_size = 0;  
    int size = 0;  
    int cnt=0;  
    int offset=0;  
  
    //FILE *myout=fopen("output_log.txt","wb+");  
    FILE *myout=stdout;  
  
    unsigned char *aacframe=(unsigned char *)malloc(1024*5);  
    unsigned char *aacbuffer=(unsigned char *)malloc(1024*1024);  
  
    FILE *ifile = fopen(url, "rb");  
    if(!ifile){  
        printf("Open file error");  
        return -1;  
    }  
  
    printf("-----+- ADTS Frame Table -+------+\n");  
    printf(" NUM | Profile | Frequency| Size |\n");  
    printf("-----+---------+----------+------+\n");  
  
    while(!feof(ifile)){  
        data_size = fread(aacbuffer+offset, 1, 1024*1024-offset, ifile);  
        unsigned char* input_data = aacbuffer;  
  
        while(1)  
        {  
            int ret=getADTSframe(input_data, data_size, aacframe, &size);  
            if(ret==-1){  
                break;  
            }else if(ret==1){  
                memcpy(aacbuffer,input_data,data_size);  
                offset=data_size;  
                break;  
            }  
  
            char profile_str[10]={0};  
            char frequence_str[10]={0};  
  
            unsigned char profile=aacframe[2]&0xC0;  
            profile=profile>>6;  
            switch(profile){  
            case 0: sprintf(profile_str,"Main");break;  
            case 1: sprintf(profile_str,"LC");break;  
            case 2: sprintf(profile_str,"SSR");break;  
            default:sprintf(profile_str,"unknown");break;  
            }  
  
            unsigned char sampling_frequency_index=aacframe[2]&0x3C;  
            sampling_frequency_index=sampling_frequency_index>>2;  
            switch(sampling_frequency_index){  
            case 0: sprintf(frequence_str,"96000Hz");break;  
            case 1: sprintf(frequence_str,"88200Hz");break;  
            case 2: sprintf(frequence_str,"64000Hz");break;  
            case 3: sprintf(frequence_str,"48000Hz");break;  
            case 4: sprintf(frequence_str,"44100Hz");break;  
            case 5: sprintf(frequence_str,"32000Hz");break;  
            case 6: sprintf(frequence_str,"24000Hz");break;  
            case 7: sprintf(frequence_str,"22050Hz");break;  
            case 8: sprintf(frequence_str,"16000Hz");break;  
            case 9: sprintf(frequence_str,"12000Hz");break;  
            case 10: sprintf(frequence_str,"11025Hz");break;  
            case 11: sprintf(frequence_str,"8000Hz");break;  
            default:sprintf(frequence_str,"unknown");break;  
            }  
  
  
            fprintf(myout,"%5d| %8s|  %8s| %5d|\n",cnt,profile_str ,frequence_str,size);  
            data_size -= size;  
            input_data += size;  
            cnt++;  
        }     
  
    }  
    fclose(ifile);  
    free(aacbuffer);  
    free(aacframe);  
  
    return 0;  
}  
```

调用：

```
simplest_aac_parser("nocturne.aac");  
```

## 结果

本程序的输入为一个AAC原始码流（裸流）的文件路径，输出为该码流中ADTS frame的统计数据，如下图所示。

![](http://img.blog.csdn.net/20160118101750487)

## 参考

* [ 视音频数据处理入门：AAC音频码流解析](http://blog.csdn.net/leixiaohua1020/article/details/50535042)