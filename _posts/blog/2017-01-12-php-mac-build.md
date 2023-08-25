---
layout: post
title: 在 MAC OSX 下搭建php开发环境
description: 详细介绍在mac平台上开发php的环境搭建。包括：搭建apache环境；安装mysql；安装phpmyadmin
category: blog
tag: php,mac php，mysql mac,phpmyadmin mac,apache mac
---

最近比较痴迷于小型的网站，这些小型的网站用php开发非常的简单迅速，特学习之。大学的时候看过php的书籍，我在此再次整理记录，并且会
在最后会开发出一个功能不多但能真实使用的php网站，并将其部署到阿里云上。

读大学时，我是在windows平台上开发php的，搭建开发环境是比较简单的，在此不做记录。现在使用的是mac平台，下面我就介绍一下如何在mac平台上搭建php开发环境。

## 配置apach服务

### 开启系统的apach服务

mac上已经自带了`apach`服务了，我们可以通过下面简单的命令来启动它：

* 开启  sudo apachectl start
* 停止  sudo apachectl stop
* 重启  sudo apachectl restart
* 查看版本  httpd -v

当开启了`apach`服务之后，在浏览器输入`localhost`之后，你将会看到具体的内容。

程序的根目录在 `/Library/WebServer/Documents/` 下，页面上显示的 “It works” 就是里面的 info.php 打印出来的。

### 更改WebServer目录到自己的根目录下

1. 在自己的用户目录下新建一个 `Sites` 文件夹。我是用户`qizhang`
2. 进到 `cd /etc/apache2/users/` 目录下， 使用 `sudo vim username.conf`，并在 username.conf中添加一下内容：

		<Directory "/Users/qizhang/Sites">
		AllowOverride All
		allow from all
		Options Indexes MultiViews FollowSymLinks
		Require all granted
		</Directory>
3. 修改`username.conf`的权限为 `644` 具体命令： `sudo chmod 644 username.conf`
4. 进到`/etc/apache2/`目录，`sudo vim httpd.conf` 将下面三句话的注释去掉：
	 
		LoadModule authz_core_module libexec/apache2/mod_authz_core.so 
		LoadModule authz_host_module libexec/apache2/mod_authz_host.so 
		LoadModule userdir_module libexec/apache2/mod_userdir.so 
前两句应该已经不带注释了，把第三句注释放开。然后找到`Include /private/etc/apache2/extra/httpd-userdir.conf` 注释放开。
5. 进到`/etc/apache2/extra/`目录 命令：`sudo vim httpd-userdir.conf ` 。 将`Include /private/etc/apache2/users/*.conf` 这句话放开注释。

然后终端输入：`sudo apachectl restart` 重启apache，浏览器输入： `loacalhost/~qizhang/` 就能看到效果了。

## 配置php

PHP的配置非常简单，就一个事，进到`/etc/apache2/`目录，编辑`httpd.conf`，找到`LoadModule php5_module libexec/apache2/libphp5.so`将其放开注释就行了。

然后`sudo apachectl restart` 重启，在用户目录的Sites文件夹下，新建一个index.php,里面`echo phpinfo(); `，就可以看到效果了。

## 安装mysql

### 安装及添加环境变量

在官网下载mysql.dmg安装包，然后双击安装。注意不需要勾选 `StartUp Item` 这一选项。

![](http://img.blog.csdn.net/20150416140840160)

在默认情况下，我们用`mysql`命令每次都要输入全路径 `sudo /usr/local/mysql/support-files/mysql.server start` 开启mysql服务; 使用 `/usr/local/mysql/bin/mysql -v` 查看mysql版本。 所以得先把bin目录配到环境变量里。具体设置环境变量的方法我在以前的博客中有总结。

### 修改root用户的密码

安装完mysql之后，默认root用户没有密码。如果你想要为它设置一个密码，就使用如下的方法。

安装完成之后，设置root用户的密码： `/usr/local/mysql/bin/mysqladmin -u root password NEW_PASSWORD_HEAR`

如果设置了之后，没有设置成功；或者以后你想修改root用户的密码，则可以通过以下命令：

	cd /usr/local/mysql/bin/
	./mysql -u root -p
	> Enter password: [type old password invisibly]
	
	use mysql;
	update user set password=PASSWORD("NEW_PASSWORD_HERE") where User='root';
	flush privileges;
	quit

## 安装phpmyadmin

可以从 <http://www.phpmyadmin.net/home_page/downloads.php> 下载.版本任意。将其解压。然后最外层文件夹名字修改为phpmyadmin，进到`~/Sites/phpmyadmin`这个目录，新建文件夹：`mkdir config` 修改读写权限：`chmod o+w config` 

然后浏览器输入：<http://localhost/~qizhang/phpmyadmin/setup/>

![](http://img.blog.csdn.net/20150416142429406)

点击 “新建服务器“，我上面已经新建好了，然后在这个界面： 

![](http://img.blog.csdn.net/20150416142746609)

密码处输入mysql的root用户密码。然后点击”应用”，记得在如下界面点击保存按钮这样config文件夹下就生成了`config.inc.php`，将该文件拷贝到phpmyadmin的根目录下。 

然后删除整个config文件夹。输入<http://localhost/~qizhang/phpmyadmin/> 就可以看到登陆phpmyadmin的界面了。 

在登陆界面，可以使用用户名为root，密码就是你mysql中root用户的密码。

注意：（如果使用root账号没有登录成功，此时密码又没有错误，可能通过以下方法解决）

将config.inc.php 中的 host的值， localhost 改为127.0.0.1

