---
layout: post
title: Mac中hosts文件和环境变量修改
description: 介绍如何修改mac的host文件和环境变量
category: blog
tag: mac,hosts,$path,环境变量
---

## 修改hosts文件

### hosts文件的作用

Hosts是一个没有扩展名的系统文件，其基本作用就是将一些常用的网址域名与其对应的IP地址建立一个关联“数据库”，当用户在浏览器中输入一个需要登录的网址时，系统会首先自动从Hosts文件中寻找对应的IP地址，一旦找到，系统会立即打开对应网页，如果没有找到，则系统再会将网址提交DNS域名解析服务器进行IP地址的解析，如果发现是被屏蔽的IP或域名，就会禁止打开此网页！

hosts文件在windows中的路径是%SystemRoot%\system32\drivers\etc\hosts，一般来说也就是C:\WINDOWS\system32\drivers\etc\hosts。

### 修改mac下的hosts文件

hosts文件在mac中的路径是 `/ect/hosts` ,但是有时候我们没有权限修改它，所以此时需要你切换到root用户去修改它。

mac上切换到root用户所需要输入的密码就是你的apple账号的密码。

## 添加环境变量

### 确定你系统 SHELL 类型

首先我们要知道你使用的 Mac OS X 是什么样的 Shell,可以通过命令 `echo $SHELL` 来查看

	zhangqis-Mac-mini:~ qizhang$ echo $SHELL
	/bin/bash
	zhangqis-Mac-mini:~ qizhang$ 

* 如果输出的是： csh 或者是 tcsh ，那么系统就是 C Shell
* 如果输出的是：bash,sh,zsh，那么系统就可能是 Bource Shell 的一个变种
* Mac OS X 10.2 之前默认是 C Shell
* Mac OS X 10.3 之后默认是 Bource Shell

如果是 Bourne Shell,那么你可以把你要添加的环境变量添加到你主目录下面的 .profile 或者 bash_profile，如果存在没有关系，添加进去即可。

### Mac配置环境变量

#### 说明



* /etc/profile   （建议不修改这个文件 ） 全局（公有）配置，不管是哪个用户，登录时都会读取该文件。
* /etc/bashrc    （一般在这个文件中添加系统级环境变量） 全局（公有）配置，bash shell执行时，不管是何种方式，都会读取此文件。
* .~/.bash_profile  （一般在这个文件中添加用户级环境变量） 每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次!

根须系统 Shell 类型不同，也有可能是以下这些：

	/etc/profile
	/etc/paths 
	~/.bash_profile 
	~/.bash_login 
	~/.profile 
	~/.bashrc

#### 设置 PATH 的语法

`export PATH="$PATH:<PATH 1>:<PATH 2>:<PATH 3>:...:<PATH N>"` 中间用冒号隔开。

查看 PATH 环境变量

`echo $PATH`


#### 注意点

* 一般环境变量更改后，重启后才可生效。如果想立刻生效，则可执行下面的语句： `source 相应文件`
* 如果默认shell是bash，那么shell启动时会触发.bashrc，如果默认shell是zsh，那么shell启动时会触发.zshrc
* 环境变量既可以加到$PATH头部，也可以加到$PATH尾部。例如mac中自带emacs的位置在/usr/local/emacs

		$ type emacs
		emacs is /usr/local/emacs
		
如果在.zshrc中添加export PATH="$PATH:/Applications/Emacs.app/Contents/MacOS"且 source .zshrc之后，type emacs还是emacs is /usr/local/emacs

解决方案是，把路径加在$PATH头部:

	export PATH="/Applications/Emacs.app/Contents/MacOS:$PATH"

或者增加alias

	alias emacs="/Applications/Emacs.app/Contents/MacOS/Emacs"


## 参考

* [mac OS X修改环境变量](http://www.jianshu.com/p/e6396fab1879)
* [Mac 可设置环境变量的位置、查看和添加PATH环境变量](http://elf8848.iteye.com/blog/1582137)