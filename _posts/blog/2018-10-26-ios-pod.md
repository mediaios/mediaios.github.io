---
layout: post
title: 发布自己的库到cocoapods
description: 发布自己的SDK到cocoapods，让别人可以使用cocoapods来使用你的库
category: blog
tag: cocoapods
---

## 关于cocoapods

cocoapods可以帮助我们集中管理使用项目中用到的第三方库。 

### 安装


1.安装 Ruby 环境。CocoaPods 是使用 Ruby 实现的，可以通过 gem 命令来安装，Mac OS X 中一般自带 Ruby 环境。接下来将默认的 RubyGems 替换为淘宝的 RubyGems 镜像，速度要快很多。

```
$ sudo gem sources -a https://ruby.taobao.org/
$ sudo gem sources -r https://rubygems.org/
$ sudo gem sources -l
```

2.安装Cocoapods

```
$ sudo gem update
$ sudo gem install -n /usr/local/bin cocoapods -v 0.39
$ pod setup
$ pod --version
```

### 在项目中使用cocoapods

在项目目录下，创建`Podfile`文件，输入一下内容：(QiTestSDKLib 是项目的target， `QiTestLib` 是一个库) 

```
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, "8.0"
target "QiTestSDKLib" do
	pod 'QiTestLib', '1.0.1'
end

```

在命令行中安装第三方库：(XCode先关闭)

```
pod install
```

成功安装后，会生成一个 .xcworkspace文件，用`XCode`打开这个文件即可。 


## 把自己写的库发布到Cocoapods

我们在项目中经常使用别人写的库，那么我们如何自己写一个库让别人来用呢？ 下面我们一步步实现。 

### 注册trunk

```
pod trunk register xxx@gmail.com 'you name' --verbose
```

注册成功后你回收到一份确认邮件。 我们可以在命令行中输入一下命令查看是否注册成功：

```
pod trunk me
```

![](https://ws1.sinaimg.cn/large/006tNbRwly1fwlnfhw4u3j30ef045q3a.jpg)

### 创建项目并上传到github

创建一个 `framework` 静态库项目，并上传到`github`。 提交代码到github时，要打上`tag`版本号，因为`Cocoapods`是根据`tag`来分析的。 

```
git tag '1.0.1'
git push --tags
git push origin master
```

### 创建podspec文件

可以利用`pod`命令创建`podspec`文件。

```
pod spec create QiTestLib
```

会生成一个`QiTestLib.podspec`文件，然后我们修改此文件成如下： 

```
#
#  Be sure to run `pod spec lint QiTestLib.podspec' to ensure this is a
#  valid spec and to remove all comments including this before submitting the spec.
#
#  To learn more about Podspec attributes see http://docs.cocoapods.org/specification.html
#  To see working Podspecs in the CocoaPods repo see https://github.com/CocoaPods/Specs/
#

Pod::Spec.new do |s|

  # ―――  Spec Metadata  ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  These will help people to find your library, and whilst it
  #  can feel like a chore to fill in it's definitely to your advantage. The
  #  summary should be tweet-length, and the description more in depth.
  #

  s.name         = "QiTestLib"
  s.version      = "1.0.1"
  s.summary      = "A SDK Demo"

  # This description is used to generate tags and improve search results.
  #   * Think: What does it do? Why did you write it? What is the focus?
  #   * Try to keep it short, snappy and to the point.
  #   * Write the description between the DESC delimiters below.
  #   * Finally, don't worry about the indent, CocoaPods strips it!
  s.description  = <<-DESC
                    "This is a project provide data calculator for ios developer"
                   DESC

  s.homepage     = "http://github.com/MaxwellQi/QiTestLib"
  # s.screenshots  = "www.example.com/screenshots_1.gif", "www.example.com/screenshots_2.gif"


  # ―――  Spec License  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Licensing your code is important. See http://choosealicense.com for more info.
  #  CocoaPods will detect a license file if there is a named LICENSE*
  #  Popular ones are 'MIT', 'BSD' and 'Apache License, Version 2.0'.
  #

  # s.license      = "MIT (example)"
   s.license      = { :type => "MIT", :file => "FILE_LICENSE" }


  # ――― Author Metadata  ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the authors of the library, with email addresses. Email addresses
  #  of the authors are extracted from the SCM log. E.g. $ git log. CocoaPods also
  #  accepts just a name if you'd rather not provide an email address.
  #
  #  Specify a social_media_url where others can refer to, for example a twitter
  #  profile URL.
  #

  s.author             = { "ethan.zhang" => "ethan.zhang@ucloud.cn" }
  # Or just: s.author    = "ethan.zhang"
  # s.authors            = { "ethan.zhang" => "ethan.zhang@ucloud.cn" }
  # s.social_media_url   = "http://twitter.com/ethan.zhang"

  # ――― Platform Specifics ――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If this Pod runs only on iOS or OS X, then specify the platform and
  #  the deployment target. You can optionally include the target after the platform.
  #

  # s.platform     = :ios
    s.platform     = :ios, "8.0"

  #  When using multiple platforms
  # s.ios.deployment_target = "5.0"
  # s.osx.deployment_target = "10.7"
  # s.watchos.deployment_target = "2.0"
  # s.tvos.deployment_target = "9.0"


  # ――― Source Location ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Specify the location from where the source should be retrieved.
  #  Supports git, hg, bzr, svn and HTTP.
  #

  s.source       = { :git => "https://github.com/MaxwellQi/QiTestLib.git", :tag => "#{s.version}" }


  # ――― Source Code ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  CocoaPods is smart about how it includes source code. For source files
  #  giving a folder will include any swift, h, m, mm, c & cpp files.
  #  For header files it will include any header in the folder.
  #  Not including the public_header_files will make all headers public.
  #

  s.source_files  = "QiTestLib/**/*.{h,m,mm}"
  #s.exclude_files = "Classes/Exclude"

  # s.public_header_files = "Classes/**/*.h"


  # ――― Resources ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  A list of resources included with the Pod. These are copied into the
  #  target bundle with a build phase script. Anything else will be cleaned.
  #  You can preserve files from being cleaned, please don't preserve
  #  non-essential files like tests, examples and documentation.
  #

  # s.resource  = "icon.png"
  # s.resources = "Resources/*.png"

  # s.preserve_paths = "FilesToSave", "MoreFilesToSave"


  # ――― Project Linking ―――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  Link your library with frameworks, or libraries. Libraries do not include
  #  the lib prefix of their name.
  #

  # s.framework  = "SomeFramework"
  # s.frameworks = "SomeFramework", "AnotherFramework"

  # s.library   = "iconv"
  # s.libraries = "iconv", "xml2"


  # ――― Project Settings ――――――――――――――――――――――――――――――――――――――――――――――――――――――――― #
  #
  #  If your library depends on compiler flags you can set them in the xcconfig hash
  #  where they will only apply to your library. If you depend on other Podspecs
  #  you can include multiple dependencies to ensure it works.

    s.requires_arc = true

  # s.xcconfig = { "HEADER_SEARCH_PATHS" => "$(SDKROOT)/usr/include/libxml2" }
  # s.dependency "JSONKit", "~> 1.4"

end

```

podspec文件中各个字段含义在此不具体说明。 

### 发布

确保库的源码在github上，并且要创建一个tag，和`podspec`里面的`s.version`保持一致。 

然后我们就可以发布了，用以下命令： 

```
pod lib lint
```

发布的同时还能看详细信息：

```
pod lib lint --verbose
```

如果发生了错误，那么用一下命令发布，它能定位错误信息： 

```
pod lib lint --no-clean
```

发布成功后，会出现如下界面： 

![](https://ws4.sinaimg.cn/large/006tNbRwly1fwlpdy91hcj30gk03ajrj.jpg)


