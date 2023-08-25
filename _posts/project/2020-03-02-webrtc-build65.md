---
layout: post
title:  编译WebRTC固定分支65
description: 下载WebRTC的65分支代码并编译出动态库——WebRTC.framework
category: project
tag: WebRTC,framework
---

## 说明 

`WebRTC`源码的下载在上一篇文章中已经介绍，此处重点介绍如何编译`WebRTC`。 

下载完源码后，首先看一下一共有哪些分支，然后再切换到你要用的分支下。 

```
git branch -r 

# 我这里使用的是65分支
git checkout -b 65 refs/remotes/branch-heads/65

git log 

cd ..

# bccbc278e6b809b0b39f4b46e6f5a072f4123182 为65分支最后一次的提交ID
gclient sync -r bccbc278e6b809b0b39f4b46e6f5a072f4123182
```

上述命令执行的过程中，会出现以下错误信息： 

```
ethan-wifi:src zhangqi$ git branch
* 65
  master
ethan-wifi:src zhangqi$ gclient sync -r bccbc278e6b809b0b39f4b46e6f5a072f4123182 --force 
Syncing projects: 100% (32/32), done.                                    

WARNING: 'src/third_party/nasm' is no longer part of this client.
It is recommended that you manually remove it or use 'gclient sync -D' next time.

WARNING: 'src/third_party/harfbuzz-ng/src' has been moved from DEPS to a higher level checkout. The git folder containing all the local branches has been saved to /Users/zhangqi/Desktop/google/webrtc/old_src_third_party_harfbuzz-ng_src.git.
If you don't care about its state you can safely remove that folder to free up space.

WARNING: 'src/third_party/freetype/src' is no longer part of this client.
It is recommended that you manually remove it or use 'gclient sync -D' next time.

________ running 'vpython src/build/landmines.py --landmine-scripts src/tools_webrtc/get_landmines.py --src-dir src' in '/Users/zhangqi/Desktop/google/webrtc'
Clobbering due to:
--- old_landmines	Sun Mar  1 17:02:53 2020
+++ new_landmines	Sun Mar  1 17:06:54 2020
@@ -8 +7,0 @@
-Clobber to change neteq_rtpplay type to executable
Running hooks:  26% ( 7/26) clang                         
________ running 'vpython src/tools/clang/scripts/update.py' in '/Users/zhangqi/Desktop/google/webrtc'
Updating Clang to 321529-1...
Creating directory /Users/zhangqi/Desktop/google/webrtc/src/third_party/llvm-build
Downloading prebuilt clang
Downloading https://commondatastorage.googleapis.com/chromium-browser-clang/Mac/clang-321529-1.tgz Traceback (most recent call last):
  File "src/tools/clang/scripts/update.py", line 944, in <module>
    sys.exit(main())
  File "src/tools/clang/scripts/update.py", line 940, in main
    return UpdateClang(args)
  File "src/tools/clang/scripts/update.py", line 488, in UpdateClang
    DownloadAndUnpackClangPackage(sys.platform)
  File "src/tools/clang/scripts/update.py", line 454, in DownloadAndUnpackClangPackage
    DownloadAndUnpack(cds_full_url, LLVM_BUILD_DIR, path_prefix)
  File "src/tools/clang/scripts/update.py", line 143, in DownloadAndUnpack
    DownloadUrl(url, f)
  File "src/tools/clang/scripts/update.py", line 102, in DownloadUrl
    response = urllib2.urlopen(url)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 154, in urlopen
    return opener.open(url, data, timeout)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 429, in open
    response = self._open(req, data)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 447, in _open
    '_open', req)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 407, in _call_chain
    result = func(*args)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 1241, in https_open
    context=self._context)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/urllib2.py", line 1195, in do_open
    h.request(req.get_method(), req.get_selector(), req.data, headers)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 1042, in request
    self._send_request(method, url, body, headers)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 1082, in _send_request
    self.endheaders(body)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 1038, in endheaders
    self._send_output(message_body)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 882, in _send_output
    self.send(msg)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 844, in send
    self.connect()
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 1255, in connect
    HTTPConnection.connect(self)
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 824, in connect
    self._tunnel()
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 796, in _tunnel
    (version, code, message) = response._read_status()
  File "/usr/local/Cellar/python/2.7.13_1/Frameworks/Python.framework/Versions/2.7/lib/python2.7/httplib.py", line 402, in _read_status
    raise BadStatusLine(line)
httplib.BadStatusLine: ''
Error: Command 'vpython src/tools/clang/scripts/update.py' returned non-zero exit status 1 in /Users/zhangqi/Desktop/google/webrtc
```

上面的错误和以前遇到的一样，还是clang没有下载成功的问题。 

## 编译过程中遇到的问题

我这边在编译65分支的过程中遇到了各种各样的问题，下面重点记录之。

在执行下面命令之前，首先要下载好clang并解压(前面文章中已经提及)。  

### 测试相关的错误

使用 GN 来生产 Ninja 工程文件，在终端执行命令： 

```
ethan-wifi:src zhangqi$ gn gen out/ios --args='target_os="ios" target_cpu="arm64" is_debug=true'
Warning: Multiple codesigning identities match "iPhone Developer"
Warning: - B134FFD7977C6CBA8C91EBEB8D578EDE17730BFA (selected)
Warning: - 5944447BB7E22557538CCD253025E4160F16FB07
Warning: - 1A869721D19303050E958ACD91B34E1786AD61BE
Warning: - 8FA8015DC902333F208FE13925D6EE244F517A1C
Warning: - 8A2570C5A4A2BF6131545B5623B4624C91B1AB73
Warning: - 5299CE872045A0DAC778A53856ADADBAF8FDE7E9
Warning: Please use either ios_code_signing_identity or 
Warning: ios_code_signing_identity_description variable to 
Warning: control which identity is selected.

ERROR at //build/config/ios/rules.gni:296:26: Assignment had no effect.
    bundle_plugins_dir = "$bundle_contents_dir/PlugIns"
                         ^-----------------------------
You set the variable "bundle_plugins_dir" here and it was unused before it went
out of scope.
See //build/config/ios/rules.gni:824:7: whence it was called.
      create_signed_bundle(_variant.target_name) {
      ^-------------------------------------------
See //testing/test.gni:269:5: whence it was called.
    ios_app_bundle(_test_target) {
    ^-----------------------------
See //webrtc.gni:277:3: whence it was called.
  test(target_name) {
  ^------------------
See //BUILD.gn:421:3: whence it was called.
  rtc_test("rtc_unittests") {
  ^--------------------------

```

解决这个问题需要添加 设置`rtc_include_tests`为false。 即 `rtc_include_tests=false`

### gn相关错误

然后有出现了新的错误： 

![](https://raw.githubusercontent.com/mediaios/img_bed/master/rtc/build_rtc_01.png)

解决这个问题需要更新 `gn`,和`clang-format.sha1`。具体下载地址如下： 

```
# 下载gn
https://storage.googleapis.com/chromium-gn/fbba40a0900ae685f5822d03d160a413d549e056

# 将其拷贝到 src/buildtools/mac 目录下面并为gn赋予可执行权限

# 上面两个命令合并在一起： 下载并重命名 

wget --no-check-certificate https://storage.googleapis.com/chromium-gn/fbba40a0900ae685f5822d03d160a413d549e056 -O src/buildtools/mac/gn

# 赋予gn可执行 
chmod 777 gn 



# 下载下面文件然后拷贝到src/buildtools/mac目录下面并命名为 clang-format.sha1
https://storage.googleapis.com/chromium-clang-format/0679b295e2ce2fce7919d1e8d003e497475f24a3

```
接着再执行 gn gen out/ios --args='target_os="ios" target_cpu="arm64" is_debug=true' 命令，结果如下： 
![](https://raw.githubusercontent.com/mediaios/img_bed/master/rtc/build_rtc_02.png)

### 默认构造方法错误 

执行 ` ./tools_webrtc/ios/build_ios_libs.sh`脚本，出现错误如下： 

![](https://raw.githubusercontent.com/mediaios/img_bed/master/rtc/build_rtc_03.png)

解决方法： 找到对应的代码注释掉 

### clang编译器相关错误 

再次执行编译脚本，错误如下： 

![](https://raw.githubusercontent.com/mediaios/img_bed/master/rtc/build_rtc_04.png)

解决方法： 用gcc编译器，不用clang编译。具体设置方法为:`is_clang=false`


## 参考 

* [webrtc 编译linux版本的各种问题](https://blog.csdn.net/u011493488/article/details/91443043)
* [WebRTC 开发（二）源码下载与编译](https://depthlove.github.io/2019/05/02/webrtc-development-2-source-code-download-and-build/)
* [WebRTC 开发（四）源码下载与更新](https://depthlove.github.io/2019/10/18/webrtc-development-4-source-code-download-and-update/)
* [使用代理同步谷歌项目时出现文件下载失败](https://idom.me/articles/843.html)



