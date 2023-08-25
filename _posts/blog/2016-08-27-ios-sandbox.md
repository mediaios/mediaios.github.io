---
layout: post
title: IOS文件目录
description: 介绍IOS的沙盒机制，及常用操作
category: blog
tag: ios, sandbox
---

## 沙盒目录结构

APP中沙盒目录结构如图：

![](https://developer.apple.com/library/content/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/art/ios_app_layout_2x.png)

下面是对各个目录的具体解释

Directory  | Description
------------- | -------------
AppName.app  | 它是app的bundle,这个目录下包含了app和app的资源。我们不可以操作这个目录，为了防止篡改，bundle目录在安装时被签名。此目录的内容不由iTunes或iCloud备份。 但是，iTunes确实会对从App Store购买的所有应用程序执行初始同步。
Documents/  | 使用此目录来存储用户生成的内容。 这个目录的内容可以通过文件共享提供给用户。 因此，他的目录应该只包含您可能希望公开给用户的文件。该目录的内容由iTunes和iCloud备份。
Documents/Inbox | 使用此目录访问您的应用程序被外部实体要求打开的文件。 比如说，邮件程序将与您的应用程序关联的电子邮件附件放置在该目录中。 文档交互控制器也可以在其中放置文件。您的应用程序可以读取和删除此目录中的文件，但无法创建新文件或写入现有文件。 如果用户尝试编辑此目录中的文件，则在进行任何更改之前，您的应用程序必须以静默方式将其移出目录。该目录的内容由iTunes和iCloud备份。
Library/			| 这是任何非用户数据文件的文件顶级目录。你通常把文件放在几个标准子目录之一中。IOS app 通常使用'Application Support'和'Caches‘子目录；并且你还可以创建自己的子目录。Library目录的内容（Caches子目录除外）由iTunes和iCloud备份。
tem/				| 使用此目录编写临时文件，在启动应用程序之间不需要保留。不再需要的时候，你的应用程序应该从这个目录中删除文件; 但是，系统可能会在应用程序未运行时清除此目录。此目录的内容不由iTunes或iCloud备份。		

