---
layout: post
title: 通过链接启动App
description: 点击一个链接，如果手机上安装了软件A，则启动A，如果没有安装A，则去App Store中去下载A
category: blog
tag: ios,ios universal links
---

## 需求

点击一个连接，如果手机上安装了软件A，则启动A，如果没有安装A，则去App Store中去下载A .

实现以上需求的技术有两种： 1. 使用 URL Scheme    2. 使用 ios9 之后得新特性，通用连接(Universal Links)


## URL Scheme

首先是为应用添加一个 `URL Scheme`。

如图：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171116UniversalLinks/universal_links_01.png)

设置完成之后在浏览器中输入:`tvuanywhere://xxxx`，如果手机上安装了tvuanywhere,则会直接打开。如图：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171116UniversalLinks/universal_links_02.png)

当app打开后，会自动调用`Appdelegate`中的如下方法：

```
- (BOOL)application:(UIApplication *)application
            openURL:(NSURL *)url
  sourceApplication:(NSString *)sourceApplication
         annotation:(id)annotation
{
    if (!url) {
        return NO;
    }
    
    NSString *urlString = [url absoluteString];
    NSLog(@"openApp:%@",urlString);
    
    return YES;
}

```


## Universal Links

这是iOS9推出的一项功能,如果你的应用支持Universal Links(通用链接),那么就能够方便的通过传统的HTTP链接来启动APP(如果iOS设备上已经安装了你的app,不需要额外做任何判断等), 或者打开网页(iOS设备上没有安装你的app)。

Universal Links(通用链接):一条普通的http链接,例如https://yohunl.com/openApp,当你支持通用链接的时候,系统中安装了你的app,那么用户点击链接,就直接进入到你的app中了,无论你是在微信中还是在其它哪里!!! 当你没有安装的时候,你可以去到你指定的页面(你可以直接展示你原来的H5页面,也可以跳转到下载页等).也就是说,用户无需知道你是否安装了app,如果安装了,这条链接就可以进app(进入你app了,你就可以以本地原生页面去展示信息给用户了),没有安装,就直接进原来的h5页面,对用户来说,是一个无缝的过程,非常顺畅!

### universal Links的优点

* 唯一性: 不像自定义的scheme,因为它使用标准的http/https链接到你的web站点,所以它不会被其它的app所声明.另外,Custom URL scheme 因为是自定义的协议，所以在没有安装 app 的情况下是无法直接打开的，而 universal links 本身是一个 HTTP/HTTPS 链接，所以有更好的兼容性。
* 安全:当用户的手机上安装了你的app,那么iOS将去你的网站上去下载你上传上去的说明文件(这个说明文件声明了你的app可以打开哪些类型的http链接).因为只有你自己才能上传文件到你网站的根目录,所以你的网站和你的app之间的关联是安全的.
* 可变:当用户手机上没有安装你的app的时候,Universal Links也能够工作.如果你愿意,在没有安装你的app的时候,用户点击链接,会在safari中展示你网站的内容.
* 简单:一个URL链接,可以同时作用于网站和app
* 私有 其它app可以在不需要知道你的app是否安装了的情况下和你的app相互通信.

### 如何支持 universal Links

首先让`Associated Domans` enable，如图所示：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171116UniversalLinks/universal_links_05.png)

必须有一个域名,且这个域名的网站需要支持https,然后拥有网站的上传到根目录的权限(这个权限是为了上传一个apple指定的文件)

创建一个json格式的命名为apple-app-site-association文件,注意这个文件必须没有后缀名,文件名必须为apple-app-site-association

```
{
"applinks": {
"apps": [],
"details": [
{
"appID": "XXXXXXXXXX.com.tvunetworks.TVUAnywherePro",
"paths": [ "/token/*"]
},
{
"appID": "XXXXXXXXXX.com.tvunetworks.TVUAnywherePro",
"paths": [ "*" ]
}
]
}
}
```

然后就是在xcode中做一些配置，首先是打开工程中的`Associated Domains`:

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171116UniversalLinks/universal_links_03.png)

在其中的Domains中填入你想支持的域名(这里不是随便填的,是可以支持你需要的Universal Links的域名), 必须以 applinks: 为前缀 .

注意:当你打开Associated Domains后,xcode会在你的工程中添加.entitlements文件 

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171116UniversalLinks/universal_links_04.png)


### 测试

在记事本中输入链接`https://domains/token/*`，查看是否生效，如图：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171116UniversalLinks/universal_links_06.png)

用浏览器打开：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20171116UniversalLinks/universal_links_07.png)

### 工程中添加处理方法

现在用户点击某个链接,直接可以进我们的app了,但是,这不是我们的最终目的,我们的目的是要能够获取到用户进来的链接,根据链接来处理,需要展示给用户的信息。

在`appdelegate`中的下面方法中做处理：

```
- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler
{
     if ([userActivity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]) {
     
         NSURL *webpageURL = userActivity.webpageURL;
         NSString *host = webpageURL.absoluteString;
         NSString *dictValue = host;
         dispatch_async(TVUMainQueue, ^{
             [kTVUNotification postNotificationName:kDistributeMsg object:nil userInfo:@{@"Data" : dictValue}];
         });
         
     }
}
```


## 参考

* [Support Universal Links](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW1)







