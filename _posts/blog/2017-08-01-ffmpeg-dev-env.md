---
layout: post
title: ffmpeg开发环境的搭建
description: windows平台和mac平台上ffmpeg开发环境搭建
category: blog
tag: ffmpeg
---

## windows平台

在windows平台下，我们选择使用`visual studio2010`开发工具。具体步骤如下：

###  新建控制台工程

新建win32控制台应用程序

### 拷贝FFmpeg开发文件
 
* 头文件(*.h)  拷贝至项目文件夹的include子文件夹下
* 导入库文件  (*.lib)拷贝至项目文件夹lib文件夹下
* 动态库文件  (*.dll)拷贝至项目文件夹下

如果直接从官网上下载的ffmpeg开发文件，则可能还需要将MinGW安装目录中的inttypes.h,stdint.h,_mingw.h三个文件拷贝至项目文件夹的include文件夹下

### 配置开发文件

打开属性面板

* 解决方案资源管理器->右键单击项目->属性

头文件配置

* 配置属性 -> C/C++ ->常规 ->附加包含目录，输入 "include" 

导入库配置

* 配置属性 -> 链接器 -> 常规 -> 附加库目录, 输入 "lib"
* 配置属性 -> 链接器 -> 输入 -> 附加依赖项, 输入"avcodec.lib;avformat.lib;avutil.lib;avdevice.lib;avfilter.lib;postproc.lib;swresample.lib;swscale.lib"

### 测试

包含头文件

* 如果是C语言，则直接导入下面ffmpeg库即可：

` #include "libavcodec/avcodec.h"`

* 如果是C++语言，则需要使用下面代码：

``` 
#define __STDC_CONSTANT_MACROS
extern "C"
{
#include "libavcodec/avcodec.h"
};

 ```

下面是测试ffmpeg开发环境是否搭建成功的代码：

```
#include "stdafx.h"

#define __STDC_CONSTANT_MACROS
extern "C"
{
#include "libavcodec/avcodec.h"
};


int _tmain(int argc, _TCHAR* argv[])
{
	printf("%s\n",avcodec_configuration());
	printf("hello world!\n");
	return 0;
}
```

如果运行无误，则表示ffmpeg开发环境已搭建完成。

## mac平台


