---
layout: post
title: appledoc在开发中的使用
description: 讲述appledoc的应用场景，以及安装和使用
category: blog
tag: net
---

## 应用场景

我们在开发一些SDK时，需要使用SDK的人介绍我们SDK中的类以及其方法属性，此时我们需要给用户提供一份开发文档。`appledoc`就是这么一个工具，为ios工程生成项目开发文档。 

## 安装

我们在此使用命令行安装。

```
git clone git://github.com/tomaz/appledoc.git
cd appledoc
sudo sh install-appledoc.sh
```

等以上命令执行完成后，验证是否安装成功：

```
macdeiMac:~ ethan$ appledoc --version
appledoc version: 2.2.1 (build 1334)
```

如上，只要显示了版本号，那么就说明`appledoc`安装成功了，接下来就可以使用它了。

## 使用

首先要确保你工程中的注释符合`appledoc`的注释规范，这样才能生成规范的API文档。

### appledoc注释规范

#### 单行注释

```
/// 单行注释   //双斜杠的注释，在使用SDK时，不会自动提示
/*  单行注释  */
/!  当行注释  */
/** 单行注释,
 * 第二行会接上第一行 
 */     
```

在注释块内，`appledoc`支持`Markdown`，`HTML`等语法。

#### 多行注释

```
/**
@brief   方法的简介。仅对属性，方法有效。 对类、协议无效，会造成后续内容解析失败。
@param   param1  参数描述
@return  返回值
@exception  可能抛出的异常
@warning  警告描述
@see     用来知名其它相关的method或function
@sa      同@see

*/
```

注意： 

@brief、@see、@sa仅对属性、方法有效，对类、协议 无效，会造成后续内容解析失败。所以该指令不能放到类注释里。

```
@see <name>
@sa <name>
其中<name>为：
1) 当前类（或协议）中的属性或方法。（注意Objective-C方法签名的写法，一般为“方法名:参数1:参数2:⋯⋯”的格式）
2) 类（或协议）名。（注意AppleDoc不支持当前类）
3) 将@see或@sa指令放在注释的最后面，避免内容丢失
```

#### 类注释

```
/** 类简介

介绍类

支持markdown等语法

*/
```

### 生成API

生成API文档的方式有两种，一种是通过命令行，另一种是通过`XCode`. 

#### 命令行方式

进入到工程所在目录(*.codeproj 所在目录)，然后执行如下命令：

```
appledoc --project-name 工程名 --project-company 公司名 --company-id 项目唯一标识 --output 生成结果要存放的目录  path(扫描哪个路径下的类, ./ 标识当前目录下的所有，会包含子目录中的类 )
```

最终会在output的目录中生成 `docset-installed.txt`的文件，然后打开文件：

```
Documentation set was installed to Xcode!

Path: /Users/ethan/Library/Developer/Shared/Documentation/DocSets/com.ucloud.unetanalysissdk.UNetAnalysisSDK.docset
Time: 2018-10-25 07:51:57 +0000
```

然后进入到`Path`目录中就能找到生成的API文档。 

### xcode方式

这种方法是直接集成到工程中。打开工程： 

首先创建 `Aggregate Target`

在`Aggregate Target`中添加脚本

![](https://ws3.sinaimg.cn/large/006tNbRwly1fwkm1h8dftj30og0igtcr.jpg)


脚本如下(注意修改成你自己公司的名称)：

```
#appledoc Xcode script  
# Start constants  
company="公司名称";  
companyID="公司id";
companyURL="公司网址";
target="iphoneos";
#target="macosx";
outputPath="导出路径";
# End constants
/usr/local/bin/appledoc \
--project-name "${PROJECT_NAME}" \
--project-company "${company}" \
--company-id "${companyID}" \
--docset-atom-filename "${company}.atom" \
--docset-feed-url "${companyURL}/${company}/%DOCSETATOMFILENAME" \
--docset-package-url "${companyURL}/${company}/%DOCSETPACKAGEFILENAME" \
--docset-fallback-url "${companyURL}/${company}" \
--output "${outputPath}" \
--publish-docset \
--docset-platform-family "${target}" \
--logformat xcode \
--keep-intermediate-files \
--no-repeat-first-par \
--no-warn-invalid-crossref \
--exit-threshold 2 \
#下面是扫描类的路径
"${PROJECT_DIR}"
```

最后选择你创建的target,编译。生成的API文档就在 `docset-installed.txt` 里面的path路径中。

最后生成的文档如下： 

![](https://ws1.sinaimg.cn/large/006tNbRwly1fwkm6nc7kaj30tm0el0tz.jpg)

![](https://ws3.sinaimg.cn/large/006tNbRwly1fwkm6xc0u3j30s40q5n01.jpg)


