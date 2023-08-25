---
layout: post
title: 利用wireShark抓取iphone手机上的网络通信包
description: 利用WireShark过滤手机上的网络通信包，分析其上面的
category: blog
tag: ios
---
## 说明

以前利用工具抓取网络请求时都是使用`Charles`，然后越来越觉得`Charles`有些卡，并且只能过滤`HTTP`协议。但是`Charles`过滤的网络请求数据显示的非常清晰。 后来就换着使用`WireShark`来抓取手机上的网络请求。

`WireShark`功能非常强大，它能过滤几乎所有的协议，并且也很少出现卡顿的情况，体验起来确实比`Charles`要好。在介绍使用`WireShark`之前，我们首先要了解一些常见的协议：HTTP,TCP,UDP等。


## HTTP协议

### 概念

HTTP协议，即超文本传输协议(Hypertext transfer protocol)。是一种详细规定了浏览器和万维网(WWW = World Wide Web)服务器之间互相通信的规则，通过因特网传送万维网文档的数据传送协议。

HTTP是一个应用层协议，由请求和响应构成，是一个标准的客户端服务器模型。HTTP是一个无状态的协议。同一个客户端的这次请求和上次请求是没有对应关系，对http服务器来说，它并不知道这两个请求来自同一个客户端。 为了解决这个问题， Web程序引入了Cookie机制来维护状态.

在Internet中所有的传输都是通过TCP/IP进行的。HTTP协议作为TCP/IP模型中应用层的协议也不例外。HTTP协议通常承载于TCP协议之上，有时也承载于TLS或SSL协议层之上，这个时候，就成了我们常说的HTTPS。

HTTP默认的端口号为80，HTTPS的端口号为443。


### URL

URL(Uniform Resource Locator) 地址用于描述一个网络上的资源, 基本格式如下：

	schema://host[:port#]/path/.../[?query-string][#anchor]

scheme 指定低层使用的协议(例如：http, https, ftp)

host HTTP服务器的IP地址或者域名

port# HTTP服务器的默认端口是80，这种情况下端口号可以省略。如果使用了别的端口，必须指明，例如 http://www.hao2you.com:8080/

path 访问资源的路径

query-string 发送给http服务器的数据

anchor- 锚

打开一个网页需要浏览器发送很多次Request:

1. 当你在浏览器输入URL http://www.baidu.com 的时候，浏览器发送一个Request去获取 http://www.baidu.com 的html. 服务器把Response发送回给浏览器.
2. 浏览器分析Response中的 HTML，发现其中引用了很多其他文件，比如图片，CSS文件，JS文件。
3. 浏览器会自动再次发送Request去获取图片，CSS文件，或者JS文件.
4. 等所有的文件都下载成功后。 网页就被显示出来了。

### HTTP消息结构

HTTP消息分为请求和响应消息。

#### HTTP请求消息

在请求消息的结构中，包括三部分：请求行、请求头、请求体。

请求行的格式为： ``METHOD /path HTTP/version-number ``

* `METHOD`:表示请求方式，常见的有`POST`,`GET`等。
* `path` : 表示请求的资源
* `HTTP/version-number` : 表示HTTP协议的版本号

请求头内包含很多的键值对。

请求体就是我们要往服务器发送的内容。在`GET`方式下的请求体是空的。

#### HTTP响应消息

响应消息的结构和请求消息的结构答题相同，同样也分为三部分：响应行、响应头、响应体。

响应行的格式为：`HTTP/version-number status-code message`

* status-code 表示状态码

#### GET请求与POST请求的区别

Http协议定义了很多与服务器交互的方法，最基本的有4种，分别是GET,POST,PUT,DELETE. 一个URL地址用于描述一个网络上的资源，而HTTP中的GET, POST, PUT, DELETE就对应着对这个资源的查，改，增，删4个操作。 我们最常见的就是GET和POST了。GET一般用于获取/查询资源信息，而POST一般用于更新资源信息。

GET和POST的区别：
1. GET提交的数据会放在URL之后，以?分割URL和传输数据，参数之间以&相连，如EditPosts.aspx?name=test1&id=123456. POST方法是把提交的数据放在HTTP包的Body中.
2. GET提交的数据大小有限制（因为浏览器对URL的长度有限制），而POST方法提交的数据没有限制.
3. GET方式需要使用Request.QueryString来取得变量的值，而POST方式通过Request.Form来获取变量的值。
4. GET方式提交数据，会带来安全问题，比如一个登录页面，通过GET方式提交数据时，用户名和密码将出现在URL上，如果页面可以被缓存或者其他人可以访问这台机器，就可以从历史记录获得该用户的账号和密码.

#### 状态码

Response 消息中的第一行叫做状态行，由HTTP协议版本号、状态码、状态消息 ，三部分组成。状态码用来告诉HTTP客户端,HTTP服务器是否产生了预期的Response.

HTTP/1.1中定义了5类状态码， 状态码由三位数字组成，第一个数字定义了响应的类别：

1XX 提示信息 - 表示请求已被成功接收，继续处理

2XX 成功 - 表示请求已被成功接收，理解，接受

3XX 重定向 - 要完成请求必须进行更进一步的处理

4XX 客户端错误 - 请求有语法错误或请求无法实现

5XX 服务器端错误 - 服务器未能实现合法的请求

下面是一些常见的状态码：

* 200 OK 表明该请求被成功地完成，所请求的资源发送回客户端
* 302 Found 重定向，新的URL会在response 中的Location中返回，浏览器将会自动使用新的URL发出新的Request
* 304  Not Modified 代表上次的文档已经被缓存了， 还可以继续使用
* 400 Bad Request 客户端请求与语法错误，不能被服务器所理解
* 403 Forbidden 服务器收到请求，但是拒绝提供服务
* 404 Not Found 请求资源不存在（输错了URL）
* 500 Internal Server Error 服务器发生了不可预期的错误
* 503 Server Unavailable 服务器当前不能处理客户端的请求，一段时间后可能恢复正常


#### 请求头

request header 中的内容大致如图：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180327wireshark/http_01.png)

##### catch头域

catch中包含的内容不仅仅是上图中显示的内容，可能还有其它类型的内容。以下是常见的catch中的内容：

If-Modified-Since ： 把浏览器端缓存页面的最后修改时间发送到服务器去，服务器会把这个时间与服务器上实际文件的最后修改时间进行对比。如果时间一致，那么返回304，客户端就直接使用本地缓存文件。如果时间不一致，就会返回200和新的文件内容。客户端接到之后，会丢弃旧文件，把新文件缓存起来，并显示在浏览器中.

If-None-Match :  If-None-Match和ETag一起工作，工作原理是在HTTP Response中添加ETag信息。 当用户再次请求该资源时，将在HTTP Request 中加入If-None-Match信息(ETag的值)。如果服务器验证资源的ETag没有改变（该资源没有更新），将返回一个304状态告诉客户端使用本地缓存文件。否则将返回200状态和新的资源和Etag. 使用这样的机制将提高网站的性能

Pragma : 防止页面被缓存， 在HTTP/1.1版本中，它和Cache-Control:no-cache作用一模一样,Pargma只有一个用法， 例如： Pragma: no-cache

```
注意: 在HTTP/1.0版本中，只实现了Pragema:no-cache, 没有实现Cache-Control
```

Cache-Control:这个是非常重要的规则。 这个用来指定Response-Request遵循的缓存机制。各个指令含义如下: 
* Cache-Control:Public 可以被任何缓存所缓存（）
* Cache-Control:Private 内容只缓存到私有缓存中
* Cache-Control:no-cache 所有内容都不会被缓存

##### Client头域

**Accept** : 浏览器端可以接受的媒体类型。

例如： Accept: text/html 代表浏览器可以接受服务器回发的类型为 text/html 也就是我们常说的html文档,

如果服务器无法返回text/html类型的数据,服务器应该返回一个406错误(non acceptable)

通配符 * 代表任意类型

例如 Accept: `*/*` 代表浏览器可以处理所有类型,(一般浏览器发给服务器都是发这个)

**Accept-Encoding**: 浏览器申明自己接收的编码方法，通常指定压缩方法，是否支持压缩，支持什么压缩方法（gzip，deflate），（注意：这不是指字符编码）;

例如： Accept-Encoding: gzip, deflate


**Accept-Language** :  浏览器申明自己接收的语言。

语言跟字符集的区别：中文是语言，中文有多种字符集，比如big5，gb2312，gbk等等；

例如： Accept-Language: en-us

**User-Agent** : 告诉HTTP服务器， 客户端使用的操作系统和浏览器的名称和版本.

我们上网登陆论坛的时候，往往会看到一些欢迎信息，其中列出了你的操作系统的名称和版本，你所使用的浏览器的名称和版本，这往往让很多人感到很神奇，实际上，服务器应用程序就是从User-Agent这个请求报头域中获取到这些信息User-Agent请求报头域允许客户端将它的操作系统、浏览器和其它属性告诉服务器。

例如： User-Agent: Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.1; Trident/4.0; CIBA; .NET CLR 2.0.50727; .NET CLR 3.0.4506.2152; .NET CLR 3.5.30729; .NET4.0C; InfoPath.2; .NET4.0E)

**Accept-Charset** : 浏览器申明自己接收的字符集，这就是本文前面介绍的各种字符集和字符编码，如gb2312，utf-8（通常我们说Charset包括了相应的字符编码方案）；

##### Cookies/Login头域

Cookie : 最重要的header, 将cookie的值发送给HTTP 服务器 

##### Miscellaneous头域

Referer: 提供了Request的上下文信息的服务器，告诉服务器我是从哪个链接过来的，比如从我主页上链接到一个朋友那里，他的服务器就能够从HTTP Referer中统计出每天有多少用户点击我主页上的链接访问他的网站。

例如: Referer:http://translate.google.cn/?hl=zh-cn&tab=wT

##### Transport头域

Connection ： `Connection: keep-alive` 当一个网页打开完成后，客户端和服务器之间用于传输HTTP数据的TCP连接不会关闭，如果客户端再次访问这个服务器上的网页，会继续使用这一条已经建立的连接； `Connection: close` 代表一个Request完成后，客户端和服务器之间用于传输HTTP数据的TCP连接会关闭， 当客户端再次发送Request，需要重新建立TCP连接。

Host : （发送请求时，该报头域是必需的）,请求报头域主要用于指定被请求资源的Internet主机和端口号，它通常从HTTP URL中提取出来的。

例如: 我们在浏览器中输入：http://www.guet.edu.cn/index.html

浏览器发送的请求消息中，就会包含Host请求报头域，如下：

Host：http://www.guet.edu.cn

此处使用缺省端口号80，若指定了端口号，则变成：Host：指定端口号 。 

#### 响应头

response header 中的内容大致如图：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180327wireshark/http_02.png)

##### Catch头域

Date : 生成消息的具体时间和日期。

Expires : 浏览器会在指定过期时间内使用本地缓存。

例如: Expires: Tue, 08 Feb 2022 11:35:14 GMT

Vary : 

例如: Vary: Accept-Encoding

##### Cookies/Login头域

P3P : 用于跨域设置Cookie, 这样可以解决iframe跨域访问cookie的问题

例如: P3P: CP=CURa ADMa DEVa PSAo PSDo OUR BUS UNI PUR INT DEM STA PRE COM NAV OTC NOI DSP COR

Set-Cookie : 非常重要的header, 用于把cookie 发送到客户端浏览器， 每一个写入cookie都会生成一个Set-Cookie.

例如: Set-Cookie: sc=4c31523a; path=/; domain=.acookie.taobao.com 

##### Entity头域

ETag : 和If-None-Match 配合使用。实例请看上节中If-None-Match的实例）

Last-Modified :  用于指示资源的最后修改日期和时间。（实例请看上节的If-Modified-Since的实例）

Content-Type : WEB服务器告诉浏览器自己响应的对象的类型和字符集。 例如：

	Content-Type: text/html; charset=utf-8
	Content-Type:text/html;charset=GB2312
	Content-Type: image/jpeg

Content-Length : 指明实体正文的长度，以字节方式存储的十进制数字来表示。在数据下行的过程中，Content-Length的方式要预先在服务器中缓存所有数据，然后所有数据再一股脑儿地发给客户端。

Content-Encoding : WEB服务器表明自己使用了什么压缩方法（gzip，deflate）压缩响应中的对象。

Content-Language :  WEB服务器告诉浏览器自己响应的对象的语言者

##### Miscellaneous头域

Server : 指明HTTP服务器的软件信息 

X-AspNet-Version: 如果网站是用ASP.NET开发的，这个header用来表示ASP.NET的版本 

X-Powered-By:表示网站是用什么技术开发的 

##### Transport头域

Connection : `Connection: keep-alive` 当一个网页打开完成后，客户端和服务器之间用于传输HTTP数据的TCP连接不会关闭，如果客户端再次访问这个服务器上的网页，会继续使用这一条已经建立的连接;`Connection: close` 代表一个Request完成后，客户端和服务器之间用于传输HTTP数据的TCP连接会关闭， 当客户端再次发送Request，需要重新建立TCP连接。

##### Location头域

Location :  用于重定向一个新的位置, 包含新的URL地址

#### HTTP协议是无状态的和Connection: keep-alive的区别

无状态是指协议对于事务处理没有记忆能力，服务器不知道客户端是什么状态。从另一方面讲，打开一个服务器上的网页和你之前打开这个服务器上的网页之间没有任何联系

HTTP是一个无状态的面向连接的协议，无状态不代表HTTP不能保持TCP连接，更不能代表HTTP使用的是UDP协议（无连接）

从HTTP/1.1起，默认都开启了Keep-Alive，保持连接特性，简单地说，当一个网页打开完成后，客户端和服务器之间用于传输HTTP数据的TCP连接不会关闭，如果客户端再次访问这个服务器上的网页，会继续使用这一条已经建立的连接

Keep-Alive不会永久保持连接，它有一个保持时间，可以在不同的服务器软件（如Apache）中设定这个时间。 




## TCP协议



## WireShark抓包

### 方法

手机连接上电脑，然后在命令行中输入以下命令来创建一个网络监听接口：
```
rvictl -s uuid
```
* uuid是设备的唯一识别号。

eg: 创建一个rvi0监听手机的网络活动

```
qis-Mac-mini:Desktop qi$ rvictl -s 6f2eb102cd69721a1456d0bca56ad0d0a23b6e89

Starting device 6f2eb102cd69721a1456d0bca56ad0d0a23b6e89 [SUCCEEDED] with interface rvi0
```

### 抓取HTTP包

抓取post请求方式的包：
![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180327wireshark/http_03.png)

抓取get请求方式的包:
![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180327wireshark/http_04.png)


## HTTPS

### 概念

HTTPS(全称：Hypertext Transfer Protocol over Secure Socket Layer)安全套接字层超文本传输协议HTTPS,为了数据传输的安全，HTTPS在HTTP的基础上加入了SSL协议，SSL依靠证书来验证服务器的身份，并为浏览器和服务器之间的通信加密。HTTPS所用的端口号是443 。 

为了能更好的理解HTTPS，首先需要了解一些概念。

### 对称加密

加密和解密使用的秘钥是相同的秘钥。因此对称加密要保证安全性的话，秘钥要做好保密，只能让使用的人知道，不可以公开。

### 非对称加密(RSA)

RSA是一种公钥密码体制，现在使用的很广泛。公钥公开，私钥保密，他的加密解密算法是公开的。由公钥加密的内容可以并且只能有私钥解密，并且由私钥加密的内容可以并且只能由公钥解密。也就是说，RSA的这一对公钥、私钥都可以用来加密和解密，并且一方加密的内容可以由并且只能由对方进行解密。可用于验证HTTPS各种秘钥的加密。

### 签名(数字签名)

签名就是在信息的后面再加上一段内容，可以证明信息没有被修改过，怎么样可以达到这个效果呢？一般是对信息做一个hash计算得到一个hash值，注意，这个过程是不可逆的，也就是说无法通过hash值得出原来的信息内容。在把信息发送出去时，把这个hash值加密后做为一个签名和信息一起发出去。 接收方在收到信息后，会重新计算信息的hash值，并和信息所附带的hash值(解密后)进行对比，如果一致，就说明信息的内容没有被修改过，因为这里hash计算可以保证不同的内容一定会得到不同的hash值，所以只要内容一被修改，根据信息内容计算的hash值就会变化。当然，不怀好意的人也可以修改信息内容的同时也修改hash值，从而让它们可以相匹配，为了防止这种情况，hash值一般都会加密后(也就是签名)再和信息一起发送，以保证这个hash值不被修改。但是客户端如何解密呢？这就涉及到数字证书了。

### 证书

证书中包含个人基本信息，以及最重要的公钥。

### 数字证书CA

如果证书在传输的过程中被篡改了呢？所以后来又出现了数字签名。

CA就是用它的私钥对消息摘要加密，形成签名，并把原始信息和数据签名进行合并，即所谓的数字证书。这样，当别人把他的证书发过来的时候，我再用同样的Hash算法，再次生成消息摘要。然后用CA的公钥对数字签名解密，得到CA创建的消息摘要，两者一比较，就知道中间有没有被人篡改了。


### HTTPS通信过程

下图是HTTPS的通信过程：

![](https://raw.githubusercontent.com/MaxwellQi/ios_workImage/master/20180327wireshark/https_01.png)

1. 客户端向服务器发送https请求
2. 服务器端返回数字证书(CA)
3. 客户端利用自己的CA[主流的CA机构证书一般都内置在各个主流浏览器中]公钥去解密证书,如果证书有问题会提示风险 
4. 如果证书没问题客户端会生成一个对称加密的随机秘钥然后再和刚刚解密的服务器端的公钥对数据进行加密，然后发送给服务器端。
5. 服务器收到以后会用自己的私钥对客户端发来的对称密钥进行解密
6. 之后双方就拿这个对称加密秘钥来进行正常通信。



## 参考
* [HTTP协议](https://hit-alibaba.github.io/interview/basic/network/HTTP.html)
* [iOS中的Socket编程](https://juejin.im/post/5958e81ef265da6c2d2c5f7f)
* [深入理解 https 通信加密过程 ](https://klionsec.github.io/2017/07/31/https-learn/)
