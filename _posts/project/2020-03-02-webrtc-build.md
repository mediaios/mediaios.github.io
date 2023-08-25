---
layout: post
title:  下载WebRTC最新源码并编译动态库
description: 下载WebRTC的最新代码并编译出动态库——WebRTC.framework
category: project
tag: WebRTC,framework
---


## 安装depot_tools工具包 

下载源码的时候，要用到 depot_tools 工具包，这是 Chromium 官方推荐的工具包，具备下载、同步、编译、上传代码等功能。depot_tools 的详细介绍见 [Using depot_tools](https://www.chromium.org/developers/how-tos/depottools)

### 获取 depot_tools源码 

depot_tools 源码属于 Google 的服务，即墙外资源，在获取 depot_tools 源码前，先需要开启 VPN 服务，然后在终端执行命令。

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git

```

如果依然出现连接失败的话，那就需要检查一下你的电脑是否真正能访问google服务。 

```
curl www.google.com
```
如果发现不能访问，则需要额外配置一下。 我翻墙使用的是`ShadowSocksX`,可以通过以下步骤查看`ShadowSocksX`的Socks5配置信息。 

![](https://raw.githubusercontent.com/mediaios/img_bed/master/rtc/rtc_source_01.png)

在终端中执行一下命令： 

```
ethan-wifi:webrtc zhangqi$ export http_proxy=socks5://127.0.0.1:10808
ethan-wifi:webrtc zhangqi$ export https_proxy=socks5://127.0.0.1:10808
ethan-wifi:webrtc zhangqi$ export all_proxy=socks5://127.0.0.1:10808
```
**执行后，只对当前终端起作用。重启终端后，默认失效。**

然后再curl一下google检测是否能访问。 

接着在终端执行命令： 

```
ethan-wifi:webrtc zhangqi$ git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
Cloning into 'depot_tools'...
remote: Sending approximately 29.75 MiB ...
remote: Total 36912 (delta 25530), reused 36912 (delta 25530)
Receiving objects: 100% (36912/36912), 29.75 MiB | 246.00 KiB/s, done.
Resolving deltas: 100% (25530/25530), done.
```

### 修改环境变量 

添加环境变量，命令格式为：

```
export PATH=$PATH:/path/depot_tools
```

其中，path 为上一步通过 pwd 命令获取的 depot_tools 文件夹所在目录。

在终端执行命令： 

```
export PATH=$PATH:/Users/zhangqi/Desktop/google/webrtc/depot_tools
```

### 检测是否安装成功 

在终端执行命令： 

```
fetch --help
```

## 下载WebRTC源码 

### 下载代码

首先要预留足够大磁盘空间，最好超过10G。 然后再终端组还行一下命令： 

```
fetch --nohooks webrtc_ios
```

这条命令会执行很长时间，耐心等待后结果如下： 

```
ethan-wifi:webrtc zhangqi$ fetch --nohooks webrtc_ios 
Running: gclient root
Traceback (most recent call last):
  File "/Users/zhangqi/Desktop/google/webrtc/depot_tools/gclient.py", line 2031, in <module>
    @metrics.collector.collect_metrics('gclient recurse')
  File "/Users/zhangqi/Desktop/google/webrtc/depot_tools/metrics.py", line 246, in _decorator
    if not self.config.should_collect_metrics:
  File "/Users/zhangqi/Desktop/google/webrtc/depot_tools/metrics.py", line 122, in should_collect_metrics
    if not self.is_googler:
  File "/Users/zhangqi/Desktop/google/webrtc/depot_tools/metrics.py", line 98, in is_googler
    self._ensure_initialized()
  File "/Users/zhangqi/Desktop/google/webrtc/depot_tools/metrics.py", line 67, in _ensure_initialized
    req = urllib.urlopen(metrics_utils.APP_URL + '/should-upload')
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
Running: gclient config --spec 'solutions = [
  {
    "url": "https://webrtc.googlesource.com/src.git",
    "managed": False,
    "name": "src",
    "deps_file": "DEPS",
    "custom_deps": {},
  },
]
target_os = ["ios", "mac"]
'
WARNING: Your metrics.cfg file was invalid or nonexistent. A new one will be created.
Running: gclient sync --nohooks --with_branch_heads
1>________ running 'git -c core.deltaBaseCacheLimit=2g clone --no-checkout --progress https://webrtc.googlesource.com/src.git /Users/zhangqi/Desktop/google/webrtc/_gclient_src_liH3Oz' in '/Users/zhangqi/Desktop/google/webrtc'
1>Cloning into '/Users/zhangqi/Desktop/google/webrtc/_gclient_src_liH3Oz'...
1>remote: Sending approximately 283.24 MiB ...        
1>remote: Total 348023 (delta 261273), reused 348023 (delta 261273)        
1>Receiving objects: 100% (348023/348023), 283.24 MiB | 936.00 KiB/s, done.
1>Resolving deltas: 100% (261273/261273), done.

[0:06:00] Still working on:
[0:06:00]   src
1>Syncing projects:   0% ( 0/ 2) 
[0:06:03] Still working on:
[0:06:03]   src
Syncing projects:  15% ( 6/38) src/buildtools/third_party/libunwind/trunk
[0:07:34] Still working on:
[0:07:34]   src/base
[0:07:34]   src/build
[0:07:34]   src/ios
[0:07:34]   src/testing
[0:07:34]   src/third_party
[0:07:34]   src/tools
[0:07:34]   src/buildtools/third_party/libc++/trunk

[0:07:44] Still working on:
[0:07:44]   src/base
[0:07:44]   src/build
[0:07:44]   src/ios
[0:07:44]   src/testing
[0:07:44]   src/third_party
[0:07:44]   src/tools
[0:07:44]   src/buildtools/third_party/libc++/trunk

[0:07:54] Still working on:
[0:07:54]   src/base
[0:07:54]   src/build
[0:07:54]   src/ios
[0:07:54]   src/testing
[0:07:54]   src/third_party
[0:07:54]   src/tools
[0:07:54]   src/buildtools/third_party/libc++/trunk

[0:07:58] Still working on:
[0:07:58]   src/base
[0:07:58]   src/build
[0:07:58]   src/ios
[0:07:58]   src/testing
[0:07:58]   src/third_party
[0:07:58]   src/tools
[0:07:58]   src/buildtools/third_party/libc++/trunk
Syncing projects:  21% ( 8/38) src/testing                               
[0:09:54] Still working on:
[0:09:54]   src/base
[0:09:54]   src/build
[0:09:54]   src/ios
[0:09:54]   src/third_party
[0:09:54]   src/tools

………………

[2:05:44] Still working on:
[2:05:44]   src/third_party/catapult
[2:05:44]   src/third_party/icu
Syncing projects: 100% (38/38), done.                       
Running: git submodule foreach 'git config -f $toplevel/.git/config submodule.$name.ignore all'
Running: git config --add remote.origin.fetch '+refs/tags/*:refs/tags/*'
Running: git config diff.ignoreSubmodules all
ethan-wifi:webrtc zhangqi$ ls
depot_tools	src
ethan-wifi:webrtc zhangqi$ 
ethan-wifi:webrtc zhangqi$
```
### 与远端代码同步

执行以下命令： 

```
gclient sync
``` 

执行结果如下： 

```
ethan-wifi:src zhangqi$ gclient sync 
Syncing projects: 100% (38/38), done.                                                     
Running hooks:  45% (10/22) mac_toolchain                 
________ running 'vpython src/build/mac_toolchain.py' in '/Users/zhangqi/Desktop/google/webrtc'
Skipping Mac toolchain installation for mac
Running hooks:  54% (12/22) clang        
________ running 'vpython src/tools/clang/scripts/update.py' in '/Users/zhangqi/Desktop/google/webrtc'
Downloading https://commondatastorage.googleapis.com/chromium-browser-clang/Mac/clang-n341867-c2900381-1.tgz Traceback (most recent call last):
  File "src/tools/clang/scripts/update.py", line 383, in <module>
    sys.exit(main())
  File "src/tools/clang/scripts/update.py", line 379, in main
    return UpdatePackage(args.package)
  File "src/tools/clang/scripts/update.py", line 309, in UpdatePackage
    DownloadAndUnpackPackage(package_file, LLVM_BUILD_DIR)
  File "src/tools/clang/scripts/update.py", line 176, in DownloadAndUnpackPackage
    DownloadAndUnpack(cds_full_url, output_dir)
  File "src/tools/clang/scripts/update.py", line 148, in DownloadAndUnpack
    DownloadUrl(url, f)
  File "src/tools/clang/scripts/update.py", line 107, in DownloadUrl
    response = urlopen(url)
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

这个问题在网上查了很久，发现无论什么样的网络环境，在mac下总会报以上错误。后来查到说可以先不用管这个错误，`update.py`这个文件就是下载clang编译器的，你自己下载下来把它解压到该放的目录即可。后面会有介绍。 

## 编译WebRTC

使用GN来产生Ninja工程文件，在终端执行以下命令： 

```
gn gen out/ios --args='target_os="ios" target_cpu="arm64" is_debug=true'

```

执行结果如下： 

```
ethan-wifi:src zhangqi$ gn gen out/ios --args='target_os="ios" target_cpu="arm64" is_debug=true'
Warning: Multiple codesigning identities match "iPhone Developer"
Warning: - 8A2570C5A4A2BF6131545B5623B4624C91B1AB73 (selected)
Warning: - 5299CE872045A0DAC778A53856ADADBAF8FDE7E9
Warning: - B134FFD7977C6CBA8C91EBEB8D578EDE17730BFA
Warning: - 5944447BB7E22557538CCD253025E4160F16FB07
Warning: - 1A869721D19303050E958ACD91B34E1786AD61BE
Warning: - 8FA8015DC902333F208FE13925D6EE244F517A1C
Warning: Please use either ios_code_signing_identity or 
Warning: ios_code_signing_identity_description variable to 
Warning: control which identity is selected.

ERROR at //build/timestamp.gni:31:19: Script returned non-zero exit code.
build_timestamp = exec_script(compute_build_timestamp,
                  ^----------
Current dir: /Users/zhangqi/Desktop/google/webrtc/src/out/ios/
Command: python /Users/zhangqi/Desktop/google/webrtc/src/build/compute_build_timestamp.py default
Returned 1.
stderr:

Traceback (most recent call last):
  File "/Users/zhangqi/Desktop/google/webrtc/src/build/compute_build_timestamp.py", line 127, in <module>
    sys.exit(main())
  File "/Users/zhangqi/Desktop/google/webrtc/src/build/compute_build_timestamp.py", line 113, in main
    last_commit_timestamp = int(open(lastchange_file).read())
IOError: [Errno 2] No such file or directory: '/Users/zhangqi/Desktop/google/webrtc/src/build/util/LASTCHANGE.committime'

See //base/BUILD.gn:34:1: whence it was imported.
import("//build/timestamp.gni")
^-----------------------------
See //third_party/opus/BUILD.gn:623:7: which caused the file to be included.
      "//base",
      ^-------

```

我这边遇到了上面的错误，然后执行以下命令： 

```
./build/util/lastchange.py  build/util/LASTCHANGE 
```

前提是确保`build/util`这个目录存在,所以要先查看以下这个目录是否存在。 


### 解决clang++问题 

由于上面报的`update.py`错误，导致lc++错误。 解决此问题的具体命令如下： 

```
ethan-wifi:src zhangqi$ curl https://commondatastorage.googleapis.com/chromium-browser-clang/Mac/clang-359912-2.tgz -o third_party/llvm-build/clang.tgz 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 27.9M  100 27.9M    0     0   916k      0  0:00:31  0:00:31 --:--:--  929k
ethan-wifi:src zhangqi$ ls third_party/llvm-build/
clang.tgz
ethan-wifi:src zhangqi$ mkdir third_party/llvm-build/Release+Asserts 
ethan-wifi:src zhangqi$ tar zxvf third_party/llvm-build/clang.tgz -C third_party/llvm-build/Release+Asserts 
x bin/
x bin/clang
x bin/clang++
x bin/clang-cl
x bin/llvm-pdbutil
x bin/llvm-symbolizer
x bin/llvm-undname
x lib/
x lib/clang/
x lib/clang/9.0.0/
x lib/clang/9.0.0/aarch64-fuchsia/
x lib/clang/9.0.0/aarch64-fuchsia/lib/
x lib/clang/9.0.0/aarch64-fuchsia/lib/libclang_rt.builtins.a
x lib/clang/9.0.0/include/
x lib/clang/9.0.0/include/__clang_cuda_builtin_vars.h
x lib/clang/9.0.0/include/__clang_cuda_cmath.h
x lib/clang/9.0.0/include/__clang_cuda_complex_builtins.h
x lib/clang/9.0.0/include/__clang_cuda_device_functions.h
x lib/clang/9.0.0/include/__clang_cuda_intrinsics.h
x lib/clang/9.0.0/include/__clang_cuda_libdevice_declares.h
x lib/clang/9.0.0/include/__clang_cuda_math_forward_declares.h
x lib/clang/9.0.0/include/__clang_cuda_runtime_wrapper.h
x lib/clang/9.0.0/include/__stddef_max_align_t.h
x lib/clang/9.0.0/include/__wmmintrin_aes.h
x lib/clang/9.0.0/include/__wmmintrin_pclmul.h
x lib/clang/9.0.0/include/adxintrin.h
x lib/clang/9.0.0/include/altivec.h
x lib/clang/9.0.0/include/ammintrin.h
x lib/clang/9.0.0/include/arm64intr.h
x lib/clang/9.0.0/include/arm_acle.h
x lib/clang/9.0.0/include/arm_fp16.h
x lib/clang/9.0.0/include/arm_neon.h
x lib/clang/9.0.0/include/armintr.h
x lib/clang/9.0.0/include/avx2intrin.h
x lib/clang/9.0.0/include/avx512bitalgintrin.h
x lib/clang/9.0.0/include/avx512bwintrin.h
x lib/clang/9.0.0/include/avx512cdintrin.h
x lib/clang/9.0.0/include/avx512dqintrin.h
x lib/clang/9.0.0/include/avx512erintrin.h
x lib/clang/9.0.0/include/avx512fintrin.h
x lib/clang/9.0.0/include/avx512ifmaintrin.h
x lib/clang/9.0.0/include/avx512ifmavlintrin.h
x lib/clang/9.0.0/include/avx512pfintrin.h
x lib/clang/9.0.0/include/avx512vbmi2intrin.h
x lib/clang/9.0.0/include/avx512vbmiintrin.h
x lib/clang/9.0.0/include/avx512vbmivlintrin.h
x lib/clang/9.0.0/include/avx512vlbitalgintrin.h
x lib/clang/9.0.0/include/avx512vlbwintrin.h
x lib/clang/9.0.0/include/avx512vlcdintrin.h
x lib/clang/9.0.0/include/avx512vldqintrin.h
x lib/clang/9.0.0/include/avx512vlintrin.h
x lib/clang/9.0.0/include/avx512vlvbmi2intrin.h
x lib/clang/9.0.0/include/avx512vlvnniintrin.h
x lib/clang/9.0.0/include/avx512vnniintrin.h
x lib/clang/9.0.0/include/avx512vpopcntdqintrin.h
x lib/clang/9.0.0/include/avx512vpopcntdqvlintrin.h
x lib/clang/9.0.0/include/avxintrin.h
x lib/clang/9.0.0/include/bmi2intrin.h
x lib/clang/9.0.0/include/bmiintrin.h
x lib/clang/9.0.0/include/cetintrin.h
x lib/clang/9.0.0/include/cldemoteintrin.h
x lib/clang/9.0.0/include/clflushoptintrin.h
x lib/clang/9.0.0/include/clwbintrin.h
x lib/clang/9.0.0/include/clzerointrin.h
x lib/clang/9.0.0/include/cpuid.h
x lib/clang/9.0.0/include/cuda_wrappers/
x lib/clang/9.0.0/include/cuda_wrappers/algorithm
x lib/clang/9.0.0/include/cuda_wrappers/complex
x lib/clang/9.0.0/include/cuda_wrappers/new
x lib/clang/9.0.0/include/emmintrin.h
x lib/clang/9.0.0/include/f16cintrin.h
x lib/clang/9.0.0/include/float.h
x lib/clang/9.0.0/include/fma4intrin.h
x lib/clang/9.0.0/include/fmaintrin.h
x lib/clang/9.0.0/include/fxsrintrin.h
x lib/clang/9.0.0/include/gfniintrin.h
x lib/clang/9.0.0/include/htmintrin.h
x lib/clang/9.0.0/include/htmxlintrin.h
x lib/clang/9.0.0/include/ia32intrin.h
x lib/clang/9.0.0/include/immintrin.h
x lib/clang/9.0.0/include/intrin.h
x lib/clang/9.0.0/include/inttypes.h
x lib/clang/9.0.0/include/invpcidintrin.h
x lib/clang/9.0.0/include/iso646.h
x lib/clang/9.0.0/include/limits.h
x lib/clang/9.0.0/include/lwpintrin.h
x lib/clang/9.0.0/include/lzcntintrin.h
x lib/clang/9.0.0/include/mm3dnow.h
x lib/clang/9.0.0/include/mm_malloc.h
x lib/clang/9.0.0/include/mmintrin.h
x lib/clang/9.0.0/include/module.modulemap
x lib/clang/9.0.0/include/movdirintrin.h
x lib/clang/9.0.0/include/msa.h
x lib/clang/9.0.0/include/mwaitxintrin.h
x lib/clang/9.0.0/include/nmmintrin.h
x lib/clang/9.0.0/include/opencl-c.h
x lib/clang/9.0.0/include/pconfigintrin.h
x lib/clang/9.0.0/include/pkuintrin.h
x lib/clang/9.0.0/include/pmmintrin.h
x lib/clang/9.0.0/include/popcntintrin.h
x lib/clang/9.0.0/include/ppc_wrappers/
x lib/clang/9.0.0/include/ppc_wrappers/mmintrin.h
x lib/clang/9.0.0/include/prfchwintrin.h
x lib/clang/9.0.0/include/ptwriteintrin.h
x lib/clang/9.0.0/include/rdseedintrin.h
x lib/clang/9.0.0/include/rtmintrin.h
x lib/clang/9.0.0/include/s390intrin.h
x lib/clang/9.0.0/include/sanitizer/
x lib/clang/9.0.0/include/sanitizer/allocator_interface.h
x lib/clang/9.0.0/include/sanitizer/asan_interface.h
x lib/clang/9.0.0/include/sanitizer/common_interface_defs.h
x lib/clang/9.0.0/include/sanitizer/coverage_interface.h
x lib/clang/9.0.0/include/sanitizer/dfsan_interface.h
x lib/clang/9.0.0/include/sanitizer/hwasan_interface.h
x lib/clang/9.0.0/include/sanitizer/linux_syscall_hooks.h
x lib/clang/9.0.0/include/sanitizer/lsan_interface.h
x lib/clang/9.0.0/include/sanitizer/msan_interface.h
x lib/clang/9.0.0/include/sanitizer/netbsd_syscall_hooks.h
x lib/clang/9.0.0/include/sanitizer/scudo_interface.h
x lib/clang/9.0.0/include/sanitizer/tsan_interface.h
x lib/clang/9.0.0/include/sanitizer/tsan_interface_atomic.h
x lib/clang/9.0.0/include/sgxintrin.h
x lib/clang/9.0.0/include/shaintrin.h
x lib/clang/9.0.0/include/smmintrin.h
x lib/clang/9.0.0/include/stdalign.h
x lib/clang/9.0.0/include/stdarg.h
x lib/clang/9.0.0/include/stdatomic.h
x lib/clang/9.0.0/include/stdbool.h
x lib/clang/9.0.0/include/stddef.h
x lib/clang/9.0.0/include/stdint.h
x lib/clang/9.0.0/include/stdnoreturn.h
x lib/clang/9.0.0/include/tbmintrin.h
x lib/clang/9.0.0/include/tgmath.h
x lib/clang/9.0.0/include/tmmintrin.h
x lib/clang/9.0.0/include/unwind.h
x lib/clang/9.0.0/include/vadefs.h
x lib/clang/9.0.0/include/vaesintrin.h
x lib/clang/9.0.0/include/varargs.h
x lib/clang/9.0.0/include/vecintrin.h
x lib/clang/9.0.0/include/vpclmulqdqintrin.h
x lib/clang/9.0.0/include/waitpkgintrin.h
x lib/clang/9.0.0/include/wbnoinvdintrin.h
x lib/clang/9.0.0/include/wmmintrin.h
x lib/clang/9.0.0/include/x86intrin.h
x lib/clang/9.0.0/include/xmmintrin.h
x lib/clang/9.0.0/include/xopintrin.h
x lib/clang/9.0.0/include/xray/
x lib/clang/9.0.0/include/xray/xray_interface.h
x lib/clang/9.0.0/include/xray/xray_log_interface.h
x lib/clang/9.0.0/include/xray/xray_records.h
x lib/clang/9.0.0/include/xsavecintrin.h
x lib/clang/9.0.0/include/xsaveintrin.h
x lib/clang/9.0.0/include/xsaveoptintrin.h
x lib/clang/9.0.0/include/xsavesintrin.h
x lib/clang/9.0.0/include/xtestintrin.h
x lib/clang/9.0.0/lib/
x lib/clang/9.0.0/lib/darwin/
x lib/clang/9.0.0/lib/darwin/libclang_rt.asan_iossim_dynamic.dylib
x lib/clang/9.0.0/lib/darwin/libclang_rt.asan_osx_dynamic.dylib
x lib/clang/9.0.0/lib/darwin/libclang_rt.fuzzer_no_main_osx.a
x lib/clang/9.0.0/lib/darwin/libclang_rt.ios.a
x lib/clang/9.0.0/lib/darwin/libclang_rt.osx.a
x lib/clang/9.0.0/lib/darwin/libclang_rt.profile_iossim.a
x lib/clang/9.0.0/lib/darwin/libclang_rt.profile_osx.a
x lib/clang/9.0.0/share/
x lib/clang/9.0.0/share/asan_blacklist.txt
x lib/clang/9.0.0/share/cfi_blacklist.txt
x lib/clang/9.0.0/x86_64-fuchsia/
x lib/clang/9.0.0/x86_64-fuchsia/lib/
x lib/clang/9.0.0/x86_64-fuchsia/lib/libclang_rt.builtins.a
x buildlog.txt
x include/
x include/c++/
x include/c++/v1/
x include/c++/v1/__bit_reference
x include/c++/v1/__bsd_locale_defaults.h
x include/c++/v1/__bsd_locale_fallbacks.h
x include/c++/v1/__config
x include/c++/v1/__debug
x include/c++/v1/__errc
x include/c++/v1/__functional_03
x include/c++/v1/__functional_base
x include/c++/v1/__functional_base_03
x include/c++/v1/__hash_table
x include/c++/v1/__libcpp_version
x include/c++/v1/__locale
x include/c++/v1/__mutex_base
x include/c++/v1/__node_handle
x include/c++/v1/__nullptr
x include/c++/v1/__split_buffer
x include/c++/v1/__sso_allocator
x include/c++/v1/__std_stream
x include/c++/v1/__string
x include/c++/v1/__threading_support
x include/c++/v1/__tree
x include/c++/v1/__tuple
x include/c++/v1/__undef_macros
x include/c++/v1/algorithm
x include/c++/v1/any
x include/c++/v1/array
x include/c++/v1/atomic
x include/c++/v1/bit
x include/c++/v1/bitset
x include/c++/v1/cassert
x include/c++/v1/ccomplex
x include/c++/v1/cctype
x include/c++/v1/cerrno
x include/c++/v1/cfenv
x include/c++/v1/cfloat
x include/c++/v1/charconv
x include/c++/v1/chrono
x include/c++/v1/cinttypes
x include/c++/v1/ciso646
x include/c++/v1/climits
x include/c++/v1/clocale
x include/c++/v1/cmath
x include/c++/v1/codecvt
x include/c++/v1/compare
x include/c++/v1/complex
x include/c++/v1/complex.h
x include/c++/v1/condition_variable
x include/c++/v1/csetjmp
x include/c++/v1/csignal
x include/c++/v1/cstdarg
x include/c++/v1/cstdbool
x include/c++/v1/cstddef
x include/c++/v1/cstdint
x include/c++/v1/cstdio
x include/c++/v1/cstdlib
x include/c++/v1/cstring
x include/c++/v1/ctgmath
x include/c++/v1/ctime
x include/c++/v1/ctype.h
x include/c++/v1/cwchar
x include/c++/v1/cwctype
x include/c++/v1/deque
x include/c++/v1/errno.h
x include/c++/v1/exception
x include/c++/v1/experimental/
x include/c++/v1/experimental/__config
x include/c++/v1/experimental/__memory
x include/c++/v1/experimental/algorithm
x include/c++/v1/experimental/any
x include/c++/v1/experimental/chrono
x include/c++/v1/experimental/coroutine
x include/c++/v1/experimental/deque
x include/c++/v1/experimental/filesystem
x include/c++/v1/experimental/forward_list
x include/c++/v1/experimental/functional
x include/c++/v1/experimental/iterator
x include/c++/v1/experimental/list
x include/c++/v1/experimental/map
x include/c++/v1/experimental/memory_resource
x include/c++/v1/experimental/numeric
x include/c++/v1/experimental/optional
x include/c++/v1/experimental/propagate_const
x include/c++/v1/experimental/ratio
x include/c++/v1/experimental/regex
x include/c++/v1/experimental/set
x include/c++/v1/experimental/simd
x include/c++/v1/experimental/string
x include/c++/v1/experimental/string_view
x include/c++/v1/experimental/system_error
x include/c++/v1/experimental/tuple
x include/c++/v1/experimental/type_traits
x include/c++/v1/experimental/unordered_map
x include/c++/v1/experimental/unordered_set
x include/c++/v1/experimental/utility
x include/c++/v1/experimental/vector
x include/c++/v1/ext/
x include/c++/v1/ext/__hash
x include/c++/v1/ext/hash_map
x include/c++/v1/ext/hash_set
x include/c++/v1/fenv.h
x include/c++/v1/filesystem
x include/c++/v1/float.h
x include/c++/v1/forward_list
x include/c++/v1/fstream
x include/c++/v1/functional
x include/c++/v1/future
x include/c++/v1/initializer_list
x include/c++/v1/inttypes.h
x include/c++/v1/iomanip
x include/c++/v1/ios
x include/c++/v1/iosfwd
x include/c++/v1/iostream
x include/c++/v1/istream
x include/c++/v1/iterator
x include/c++/v1/limits
x include/c++/v1/limits.h
x include/c++/v1/list
x include/c++/v1/locale
x include/c++/v1/locale.h
x include/c++/v1/map
x include/c++/v1/math.h
x include/c++/v1/memory
x include/c++/v1/module.modulemap
x include/c++/v1/mutex
x include/c++/v1/new
x include/c++/v1/numeric
x include/c++/v1/optional
x include/c++/v1/ostream
x include/c++/v1/queue
x include/c++/v1/random
x include/c++/v1/ratio
x include/c++/v1/regex
x include/c++/v1/scoped_allocator
x include/c++/v1/set
x include/c++/v1/setjmp.h
x include/c++/v1/shared_mutex
x include/c++/v1/span
x include/c++/v1/sstream
x include/c++/v1/stack
x include/c++/v1/stdbool.h
x include/c++/v1/stddef.h
x include/c++/v1/stdexcept
x include/c++/v1/stdint.h
x include/c++/v1/stdio.h
x include/c++/v1/stdlib.h
x include/c++/v1/streambuf
x include/c++/v1/string
x include/c++/v1/string.h
x include/c++/v1/string_view
x include/c++/v1/strstream
x include/c++/v1/support/
x include/c++/v1/support/android/
x include/c++/v1/support/android/locale_bionic.h
x include/c++/v1/support/fuchsia/
x include/c++/v1/support/fuchsia/xlocale.h
x include/c++/v1/support/ibm/
x include/c++/v1/support/ibm/limits.h
x include/c++/v1/support/ibm/locale_mgmt_aix.h
x include/c++/v1/support/ibm/support.h
x include/c++/v1/support/ibm/xlocale.h
x include/c++/v1/support/musl/
x include/c++/v1/support/musl/xlocale.h
x include/c++/v1/support/newlib/
x include/c++/v1/support/newlib/xlocale.h
x include/c++/v1/support/solaris/
x include/c++/v1/support/solaris/floatingpoint.h
x include/c++/v1/support/solaris/wchar.h
x include/c++/v1/support/solaris/xlocale.h
x include/c++/v1/support/win32/
x include/c++/v1/support/win32/limits_msvc_win32.h
x include/c++/v1/support/win32/locale_win32.h
x include/c++/v1/support/xlocale/
x include/c++/v1/support/xlocale/__nop_locale_mgmt.h
x include/c++/v1/support/xlocale/__posix_l_fallback.h
x include/c++/v1/support/xlocale/__strtonum_fallback.h
x include/c++/v1/system_error
x include/c++/v1/tgmath.h
x include/c++/v1/thread
x include/c++/v1/tuple
x include/c++/v1/type_traits
x include/c++/v1/typeindex
x include/c++/v1/typeinfo
x include/c++/v1/unordered_map
x include/c++/v1/unordered_set
x include/c++/v1/utility
x include/c++/v1/valarray
x include/c++/v1/variant
x include/c++/v1/vector
x include/c++/v1/version
x include/c++/v1/wchar.h
x include/c++/v1/wctype.h
```

### 编译出WebRTC.frameowrk 

前提设置： 

* 在 `src/tools_webrtc/ios/build_ios_libs.py`中设置支持持arm64，即 `DEFAULT_ARCHS = ENABLED_ARCHS = ['arm64']`
* 在 `src/tools_webrtc/ios/build_ios_libs.py`中设置最低支持的ios系统为 9.0 
* 在`BUILD.gn`中把`-Wno-misleading-indentation`,`-Wno-implicit-int-float-conversion`,`-Wno-final-dtor-non-final-class`,`-Wno-builtin-assume-aligned-alignment`,`-Wno-deprecated-copy`这些设置全部注释掉。不然会出现错误。
*  

```
ethan-wifi:src zhangqi$ python tools_webrtc/ios/build_ios_libs.py --bitcode
INFO:root:Building WebRTC with args: target_os="ios" ios_enable_code_signing=false use_xcode_clang=true is_component_build=false is_debug=false target_cpu="arm64" ios_deployment_target="8.0" rtc_libvpx_build_vp9=false enable_ios_bitcode=true use_goma=false enable_stripping=true
Done. Made 1521 targets from 229 files in 2479ms
INFO:root:Building target: framework_objc
ninja: Entering directory `/Users/zhangqi/Desktop/google/webrtc/src/out_ios_libs/arm64_libs'
[927/2642] CXX clang_x64/obj/buildtools/third_party/libc++/libc++/stdexcept.o
FAILED: clang_x64/obj/buildtools/third_party/libc++/libc++/stdexcept.o 
../../third_party/llvm-build/Release+Asserts/bin/clang++ -MMD -MF clang_x64/obj/buildtools/third_party/libc++/libc++/stdexcept.o.d -D_LIBCPP_BUILDING_LIBRARY -DLIBCXX_BUILDING_LIBCXXABI -D_LIBCPP_HAS_NO_ALIGNED_ALLOCATION -DCR_XCODE_VERSION=1121 -DCR_CLANG_REVISION=\"n341867-c2900381-1\" -D_LIBCPP_ABI_UNSTABLE -D_LIBCPP_DISABLE_VISIBILITY_ANNOTATIONS -D_LIBCXXABI_DISABLE_VISIBILITY_ANNOTATIONS -D_LIBCPP_ENABLE_NODISCARD -DCR_LIBCXX_REVISION=375504 -D__ASSERT_MACROS_DEFINE_VERSIONS_WITHOUT_UNDERSCORES=0 -DNDEBUG -DNVALGRIND -DDYNAMIC_ANNOTATIONS_ENABLED=0 -I../.. -Iclang_x64/gen -fvisibility-global-new-delete-hidden -fno-strict-aliasing -fstack-protector -fcolor-diagnostics -fmerge-all-constants -arch x86_64 -Wno-builtin-macro-redefined -D__DATE__= -D__TIME__= -D__TIMESTAMP__= -Xclang -fdebug-compilation-dir -Xclang . -no-canonical-prefixes -O2 -fno-omit-frame-pointer -gdwarf-4 -g2 -isysroot ../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk -mmacosx-version-min=10.10.0 -fvisibility=hidden -Wheader-hygiene -Wstring-conversion -Wtautological-overlap-compare -fstrict-aliasing -fPIC -Werror -Wall -Wno-unused-variable -Wno-misleading-indentation -Wunguarded-availability -Wno-missing-field-initializers -Wno-unused-parameter -Wno-c++11-narrowing -Wno-unneeded-internal-declaration -Wno-undefined-var-template -Wno-ignored-pragma-optimize -Wno-implicit-int-float-conversion -Wno-final-dtor-non-final-class -Wno-builtin-assume-aligned-alignment -Wno-deprecated-copy -Wno-range-loop-analysis -std=c++14 -stdlib=libc++ -nostdinc++ -isystem../../buildtools/third_party/libc++/trunk/include -isystem../../buildtools/third_party/libc++abi/trunk/include -fvisibility-inlines-hidden -fexceptions -frtti -c ../../buildtools/third_party/libc++/trunk/src/stdexcept.cpp -o clang_x64/obj/buildtools/third_party/libc++/libc++/stdexcept.o
error: unknown warning option '-Wno-misleading-indentation'; did you mean '-Wno-binding-in-condition'? [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-implicit-int-float-conversion'; did you mean '-Wno-implicit-float-conversion'? [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-final-dtor-non-final-class'; did you mean '-Wno-abstract-final-class'? [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-builtin-assume-aligned-alignment' [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-deprecated-copy'; did you mean '-Wno-deprecated'? [-Werror,-Wunknown-warning-option]
[928/2642] CXX clang_x64/obj/buildtools/third_party/libc++/libc++/string.o
FAILED: clang_x64/obj/buildtools/third_party/libc++/libc++/string.o 
../../third_party/llvm-build/Release+Asserts/bin/clang++ -MMD -MF clang_x64/obj/buildtools/third_party/libc++/libc++/string.o.d -D_LIBCPP_BUILDING_LIBRARY -DLIBCXX_BUILDING_LIBCXXABI -D_LIBCPP_HAS_NO_ALIGNED_ALLOCATION -DCR_XCODE_VERSION=1121 -DCR_CLANG_REVISION=\"n341867-c2900381-1\" -D_LIBCPP_ABI_UNSTABLE -D_LIBCPP_DISABLE_VISIBILITY_ANNOTATIONS -D_LIBCXXABI_DISABLE_VISIBILITY_ANNOTATIONS -D_LIBCPP_ENABLE_NODISCARD -DCR_LIBCXX_REVISION=375504 -D__ASSERT_MACROS_DEFINE_VERSIONS_WITHOUT_UNDERSCORES=0 -DNDEBUG -DNVALGRIND -DDYNAMIC_ANNOTATIONS_ENABLED=0 -I../.. -Iclang_x64/gen -fvisibility-global-new-delete-hidden -fno-strict-aliasing -fstack-protector -fcolor-diagnostics -fmerge-all-constants -arch x86_64 -Wno-builtin-macro-redefined -D__DATE__= -D__TIME__= -D__TIMESTAMP__= -Xclang -fdebug-compilation-dir -Xclang . -no-canonical-prefixes -O2 -fno-omit-frame-pointer -gdwarf-4 -g2 -isysroot ../../../../../../../../Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.15.sdk -mmacosx-version-min=10.10.0 -fvisibility=hidden -Wheader-hygiene -Wstring-conversion -Wtautological-overlap-compare -fstrict-aliasing -fPIC -Werror -Wall -Wno-unused-variable -Wno-misleading-indentation -Wunguarded-availability -Wno-missing-field-initializers -Wno-unused-parameter -Wno-c++11-narrowing -Wno-unneeded-internal-declaration -Wno-undefined-var-template -Wno-ignored-pragma-optimize -Wno-implicit-int-float-conversion -Wno-final-dtor-non-final-class -Wno-builtin-assume-aligned-alignment -Wno-deprecated-copy -Wno-range-loop-analysis -std=c++14 -stdlib=libc++ -nostdinc++ -isystem../../buildtools/third_party/libc++/trunk/include -isystem../../buildtools/third_party/libc++abi/trunk/include -fvisibility-inlines-hidden -fexceptions -frtti -c ../../buildtools/third_party/libc++/trunk/src/string.cpp -o clang_x64/obj/buildtools/third_party/libc++/libc++/string.o
error: unknown warning option '-Wno-misleading-indentation'; did you mean '-Wno-binding-in-condition'? [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-implicit-int-float-conversion'; did you mean '-Wno-implicit-float-conversion'? [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-final-dtor-non-final-class'; did you mean '-Wno-abstract-final-class'? [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-builtin-assume-aligned-alignment' [-Werror,-Wunknown-warning-option]
error: unknown warning option '-Wno-deprecated-copy'; did you mean '-Wno-deprecated'? [-Werror,-Wunknown-warning-option]
[932/2642] CXX obj/modules/video_coding/webrtc_vp8_temporal_layers/default_temporal_layers.o
ninja: build stopped: subcommand failed.
Traceback (most recent call last):
  File "tools_webrtc/ios/build_ios_libs.py", line 239, in <module>
    sys.exit(main())
  File "tools_webrtc/ios/build_ios_libs.py", line 170, in main
    args.use_goma, gn_args)
  File "tools_webrtc/ios/build_ios_libs.py", line 142, in BuildWebRTC
    _RunCommand(cmd)
  File "tools_webrtc/ios/build_ios_libs.py", line 77, in _RunCommand
    subprocess.check_call(cmd, cwd=SRC_DIR)
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/subprocess.py", line 540, in check_call
    raise CalledProcessError(retcode, cmd)
subprocess.CalledProcessError: Command '['/Users/zhangqi/Desktop/google/webrtc/src/third_party/depot_tools/ninja', '-C', '/Users/zhangqi/Desktop/google/webrtc/src/out_ios_libs/arm64_libs', 'framework_objc']' returned non-zero exit status 1
``` 

把`-Wno-misleading-indentation`,`-Wno-implicit-int-float-conversion`,`-Wno-final-dtor-non-final-class`,`-Wno-builtin-assume-aligned-alignment`,`-Wno-deprecated-copy`设置注释掉之后，编译就会成功，具体执行过程如下： 

```
ethan-wifi:src zhangqi$ python tools_webrtc/ios/build_ios_libs.py --bitcode
INFO:root:Building WebRTC with args: target_os="ios" ios_enable_code_signing=false use_xcode_clang=true is_component_build=false is_debug=false target_cpu="arm64" ios_deployment_target="9.0" rtc_libvpx_build_vp9=false enable_ios_bitcode=true use_goma=false enable_stripping=true
Done. Made 1521 targets from 229 files in 2337ms
INFO:root:Building target: framework_objc
ninja: Entering directory `/Users/zhangqi/Desktop/google/webrtc/src/out_ios_libs/arm64_libs'
[2642/2642] STAMP obj/sdk/framework_objc.stamp
INFO:root:Merging framework slices.
INFO:root:Done.

```

查看其支持的cpu架构： 

```
ethan-wifi:src zhangqi$ lipo -info out_ios_libs/WebRTC.framework/WebRTC 
Architectures in the fat file: out_ios_libs/WebRTC.framework/WebRTC are: arm64 
```




