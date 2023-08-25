---
layout: post
title: IOS Code Signing
description: ios 签名的讲解，主要介绍如何用命令行进行代码签名
category: blog
tag: ios, Objective-C, Swift
---

## 签名相关的命令

介绍在命令行中如何使用和签名相关的命令。

### 快捷查看系统中能用来对代码进行签名的证书：

```
zhangqis-Mac-mini:~ qizhang$ security find-identity -v -p codesigning
  1) 43048DCFC9CDE6E0C6FA62EC4C81B55885DB7827 "Mac Developer: qi zhang (J3GC2Z25R4)" (CSSMERR_TP_CERT_REVOKED)
  2) B358C5B2ECD857563C7531CEE1B4C425DCF182D2 "Developer ID Application: TVU Networks Corporation (YDEVQQ22JR)"
  3) 715A5A723125DF8E9C9326B7289711FC9664243A "iPhone Developer: qi zhang (J3GC2Z25R4)" (CSSMERR_TP_CERT_REVOKED)
  4) 387F950BF557A30F0F04CBFC6400239174E970C9 "Developer ID Application: TVU Networks Corporation (YDEVQQ22JR)"
  5) D3B2036FD0E33B3357B85082B1FF48C294BF7324 "iPhone Developer: qi zhang (J3GC2Z25R4)"
  6) B6D0AA36445404737FD602909BD60E4C60BA8BC3 "iPhone Distribution: TVU Networks Corporation (YDEVQQ22JR)"
  7) 0EC8D42AFB9ACCB7BFFF144D9104590F173A7321 "Mac Developer: qi zhang (J3GC2Z25R4)" (CSSMERR_TP_CERT_REVOKED)
  8) D928823BBFA8A35D47C4E20543AD243D863BD35A "iPhone Developer: qizhang@tvunetworks.com (U8GR82KW82)"
  9) 2A3C952F99F3A5E91EBC394B2C74A3C4053AC8CD "iPhone Developer: qi zhang (J3GC2Z25R4)"
     9 valid identities found
```

### 对未签名的app手动签名

```
zhangqis-Mac-mini:~ qizhang$ codesign -s 'iPhone Developer: qi zhang (J3GC2Z25R4)' TVUAnywhere.app
```

### 对已签名app重新签名

```
zhangqis-Mac-mini:~ qizhang$ codesign -f -s 'iPhone Developer: qi zhang (J3GC2Z25R4)' TVUAnywhere.app
```

### 查看指定app的签名信息

```
zhangqis-Mac-mini:~ qizhang$ codesign -vv -d TVUAnywhere.app
```

### 验证签名文件的完整性

```
zhangqis-Mac-mini:~ qizhang$ codesign --verify TVUAnywhere.app
```

## 资源文件签名

iOS 和 OS X 的应用和框架则是包含了它们所需要的资源在其中的。这些资源包括图片和不同的语言文件，资源中也包括很重要的应用组成部分例如 XIB/NIB 文件，存档文件(archives)，甚至是证书文件。所以为一个程序包设置签名时，这个包中的所有资源文件也都会被设置签名。

为了达到为所有文件设置签名的目的，签名的过程中会在程序包（即Example.app）中新建一个叫做 _CodeSignatue/CodeResources 的文件，这个文件中存储了被签名的程序包中所有文件的签名。你可以自己去查看这个签名列表文件，它仅仅是一个 plist 格式文件。

这个列表文件中不光包含了文件和它们的签名的列表，还包含了一系列规则，这些规则决定了哪些资源文件应当被设置签名。伴随 OS X 10.10 DP 5 和 10.9.5 版本的发布，苹果改变了代码签名的格式，也改变了有关资源的规则。如果你使用10.9.5或者更高版本的 codesign 工具，在 CodeResources 文件中会有4个不同区域，其中的 rules 和 files 是为老版本准备的，而 files2 和 rules2 是为新的第二版的代码签名准备的。最主要的区别是在新版本中你无法再将某些资源文件排除在代码签名之外，在过去你是可以的，只要在被设置签名的程序包中添加一个名为 ResourceRules.plist 的文件，这个文件会规定哪些资源文件在检查代码签名是否完好时应该被忽略。但是在新版本的代码签名中，这种做法不再有效。所有的代码文件和资源文件都必须设置签名，不再可以有例外。在新版本的代码签名规定中，一个程序包中的可执行程序包，例如扩展 (extension)，是一个独立的需要设置签名的个体，在检查签名是否完整时应当被单独对待。