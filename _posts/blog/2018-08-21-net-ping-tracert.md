---
layout: post
title: 在iOS平台实现Ping和traceroute
description: ios平台网络诊断SDK，支持对ip和域名的ping,traceroute(udp,icmp协议)，支持tcp ping, 端口扫描，nslookup等功能
category: blog
tag: ios ping, ping, traceroute, ios icmp,icmp ios,icmp traceroute ios,icmp traceroute,udp traceroute,udp traceroute ios
---

## 概述

本篇文章主要讲ping和traceroute的原理以及在ios平台的实现，所有的代码都放到了github上并且发布到了cocoapods。你可以通过这个SDK非常容易的就能实现ping和traceroute(包含UDP和ICMP)等网络诊断的基本功能。

* github地址： [net-diagnosis](https://github.com/mediaios/net-diagnosis)
* 欢迎fork和star !
* 掘金地址: [在iOS平台实现Ping和traceroute](https://juejin.im/post/5c7cedaaf265da2da15ddbad) 


下面我们从以下几个方面做介绍： 

* ping命令 
* icmp协议
	* icmp技术细节
		* icmp报文结构
		* icmp报头
		* 填充数据
* ping
	* ping实现原理
	* 利用wireshark查看ping
* 计算机网络基础
	* TCP/IP协议栈与数据包封装
	* IP数据包格式
	* ping的实现(oc&c++)
		* 技术预研与构思
		* 具体实现
	* TCP ping的原理及实现
* traceroute
	* traceroute命令的原理及过程
	* whireshark查看traceroute
	* udp traceroute的实现
	* udp traceroute存在的问题
	* icmp traceroute
	* icmp traceourte的实现   



## ping 命令 

`Ping`是为了测试另一台主机是否可达，现在已经成为一种常用的网络状态检查工具。


常见的ping命令： 

```
/**** 往目的追击发送固定包数 ****/
ping -c 3 www.baidu.com   // ping百度发送3个包

/**** 设置两次发包之间的等待时间 ****/
ping -i 5 www.baidu.com   // 两包之间的时间间隔为5s
ping -i 0.1 www.baidu.com // 两包之间的时间间隔为0.1s

/**** 检查本地网络接口是否已经启动并正在运行  ****/
ping 127.0.0.1  (linux: ping 0) 
ping localhost 

/**** 超级用户可以利用 -f 几秒钟发送数十万个包给主服务造成压力 *****/
sudo ping -f www.baidu.com 

/**** 让电脑发出蜂鸣声: 响应包到达目时，会发出声音  ****/
ping -a www.baidu.com 

/**** 只打印ping的汇总结果  ****/
ping -c 5 -q www.baidu.com

/**** 修改ping包(icmp包)的大小 ****/
ping -s 100 -c 5 www.baidu.com




```

示例： 

```
macdeiMac:PhoneNetSDK ethan$ ping www.baidu.com

PING www.a.shifen.com (61.135.169.121): 56 data bytes
64 bytes from 61.135.169.121: icmp_seq=0 ttl=49 time=32.559 ms
64 bytes from 61.135.169.121: icmp_seq=1 ttl=49 time=32.413 ms
64 bytes from 61.135.169.121: icmp_seq=2 ttl=49 time=32.489 ms
^C
--- www.a.shifen.com ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 32.413/32.487/32.559/0.060 ms
macdeiMac:PhoneNetSDK ethan$ 
```

分析以上结果： 

* 发送端信息
    *  www.a.shifen.com (61.135.169.121): 对域名做了自动DNS解析
    *  56 data bytes: 向该主机发送大小是56字节的数据包。

* 主机响应的信息
    * icmp_seq:  响应包的序列号。
    * ttl: ip数据报的ttl值。
    * time:请求往返耗时。
    * 64 bytes:响应数据包的大小是64个字节。 
* 统计总结信息
    *  0.0% packet loss： 总共发了3个包丢包率是0%
    *  min/avg/max = 32.413/32.487/32.559：最小/平均/最大往返时间32.413/32.487/32.559

TTL(Time to live): IP数据报的生存时间，单位是hop(跳)。比如64，每过一个路由器就把该值减1，如果减到0 就表示路由已经太长了仍然找不到目的主机的网络，就丢弃该包。


问题：在发包时，为什么发送的是56字节的包，主机响应的却是64字节的包？ 在这里的56和64是同一个概念吗？ 


## icmp

互联网控制消息协议（英语：Internet Control Message Protocol，缩写：ICMP）是互联网协议族的核心协议之一。它是TCP/IP协议族的一个子协议，它用于TCP/IP网络中发送控制消息，提供可能发生在通信环境中的各种问题反馈，通过这些信息，使管理者可以对所发生的问题作出诊断，然后采取适当的措施解决。

控制消息有：目的不可达下次，超时信息，重定向消息，时间戳请求和时间戳响应消息，回显请求和回显应答消息。

ICMP [1]依靠IP來完成它的任务，它是IP的主要部分。它与传输协议（如TCP和UDP）显著不同：它一般不用于在两点间传输数据。它通常不由网络程序直接使用，除了ping和traceroute这两个特別的例子。 IPv4中的ICMP被称作ICMPv4，IPv6中的ICMP则被称作ICMPv6。

### icmp技术细节

CMP是在RFC 792中定义的互联网协议族之一。通常用于返回的错误信息或是分析路由。ICMP错误消息总是包括了源数据并返回给发送者。 ICMP错误消息的例子之一是TTL值过期。每个路由器在转发数据报的时候都会把IP包头中的TTL值减1。如果TTL值为0，“TTL在传输中过期”的消息将会回报给源地址。 每个ICMP消息都是直接封裝在一个IP数据包中的，因此，和UDP一样，ICMP是不可靠的。

虽然ICMP是包含在IP数据包中的，但是对ICMP消息通常会特殊处理，会和一般IP数据包的处理不同，而不是作为IP的一个子协议来处理。在很多时候，需要去查看ICMP消息的內容，然后发送过当的错误消息到那个原來产生IP数据包的程序，即那个导致ICMP信息被传送的IP数据包。

很多常用的工具是基于ICMP消息的。traceroute是通过发送包含有特殊的TTL的包，然后接收ICMP超超消息和目标不可达消息來实现的。ping则是用ICMP的”Echo request”（类别代码：8）和”Echo reply”（类别代码：0）消息來实现的。

#### icmp报文结构

#### 报头

ICMP报头从IP报头的第160位开始(ip首部20字节)

![](https://user-gold-cdn.xitu.io/2019/3/25/169b29790f817f25?w=366&h=101&f=jpeg&s=23622)

* Type: ICMP的类型，标识生成的错误报文
* Code: 进一步割分ICMP的类型，该字段用来查找产生错误的原因；例如ICMP的目标不可达类型可以把这个位设置为1-15等来表示不同的意思。
* Checksum : 校验码部分，这个字段包含有从ICMP报头和数据部分计算得来，用于检查错误的数据，其中此校验码字段的值视为0
* ID ：这个字段包含了ID值，在`Echo Reply`类型的消息中要返回这个字段
* Sequence : 这个字段包含一个序号，同样要在`Echo Reply`类型的消息中要返回这个字段

#### 填充数据

填充的数据紧接在ICMP报头的后面(以8位为一组)：

* Linux的ping工具填充的ICMP除了8个8位元组的报头以外，默认情况下还另外填充数据使得总大小位64字节。
* Windows的ping.exe填充的ICMP除了8个8位元组的报头以外，默认情况下还另外填充数据使得总大小位40字节。





## ping

### ping实现原理

`Ping`是为了测试另一台主机是否可达，现在已经成为一种常用的网络状态检查工具。该程序发送一份 ICMP回显请求报文给远程主机，并等待返回 ICMP回显应答。

ping 使用的是ICMP协议，它发送icmp回送请求消息给目的主机。ICMP协议规定：目的主机必须返回ICMP回送应答消息给源主机。如果源主机在一定时间内收到应答，则认为主机可达。大多数的 TCP/IP 实现都在内核中直接支持Ping服务器，ICMP回显请求和回显应答报文如下图所示。

![image](https://user-gold-cdn.xitu.io/2019/3/4/169480089d35ba85?w=488&h=156&f=jpeg&s=10086)


ping的原理： 

![](https://user-gold-cdn.xitu.io/2019/3/25/169b29790f2a8d93?w=632&h=188&f=jpeg&s=28097)

ping的原理是用类型码为8的ICMP发请求，收到请求的主机则用类型码为0的ICMP回应。通过计算ICMP应答报文数量和与接受与发送报文之间的时间差，判断当前的网络状态。这个往返时间的计算方法是：ping命令在发送ICMP报文时将当前的时间值存储在ICMP报文中发出，当应答报文返回时，使用当前时间值减去存放在ICMP报文数据中存放发送请求的时间值来计算往返时间。ping返回接收到的数据报文字节大小、TTL值以及往返时间。

### 利用wireshark查看ping

我在命令行中ping www.baidu.com 以下是显示结果：

![image](https://user-gold-cdn.xitu.io/2019/3/4/169480089db67290?w=1168&h=743&f=jpeg&s=192461)

 
![image](https://user-gold-cdn.xitu.io/2019/3/4/169480089e231271?w=1147&h=796&f=jpeg&s=197983)

如上图所示，icmp包的type是8 ， 是request请求； icmp的包type是0 ，是reply. 


## 计算机网络基础知识

###  TCP/IP协议栈与数据包封装

OSI七层模型以及TCP/IP模型：

![](https://user-gold-cdn.xitu.io/2019/3/25/169b29791037d912?w=804&h=432&f=jpeg&s=117678)

两台计算机通过TCP/IP的通信过程如下： 

![](https://user-gold-cdn.xitu.io/2019/3/25/169b2979365387be?w=600&h=353&f=png&s=55503)

传输层及其以下的机制由内核提供，应用层由用户进程提供,应用程序对通讯数据的含义进行解释，而传输层及其以下处理通讯的细节，将数据从一台计算机通过一定的路径发送到另一台计算机。应用层数据通过协议栈发到网络上时，每层协议都要加上一个数据首部（header），称为封装（Encapsulation）。 

TCP/IP数据包的封装： 

![](https://user-gold-cdn.xitu.io/2019/3/25/169b29793fce83b2?w=600&h=444&f=png&s=80425)

目的主机收到数据包后，经过各层协议栈最后到达应用程序。

![](https://user-gold-cdn.xitu.io/2019/3/25/169b2979366b9146?w=600&h=372&f=png&s=69584)

以太网驱动程序首先根据以太网首部中的“上层协议”字段确定该数据帧的有效载荷是IP、ARP还是RARP协议的数据报，然后交给相应的协议处理。假如是IP数据报，IP协议再根据IP首部中的“上层协议”字段确定该数据报的有效载荷是TCP、UDP、ICMP还是IGMP，然后交给相应的协议处理。假如是TCP段或UDP段，TCP或UDP协议再根据TCP首部或UDP首部的“端口号”字段确定应该将应用层数据交给哪个用户进程。IP地址是标识网络中不同主机的地址，而端口号就是同一台主机上标识不同进程的地址，IP地址和端口号合起来标识网络中唯一的进程。

注意，虽然IP、ARP和RARP数据报都需要以太网驱动程序来封装成帧，但是从功能上划分，ARP和RARP属于链路层，IP属于网络层。虽然ICMP、IGMP、TCP、UDP的数据都需要IP协议来封装成数据报，但是从功能上划分，ICMP、IGMP与IP同属于网络层，TCP和UDP属于传输层。

### IP数据报格式

IPv4数据包格式如下： 

![](https://user-gold-cdn.xitu.io/2019/3/25/169b297952f095e5?w=600&h=388&f=png&s=69145)

关于首部长度： 

根据IP数据报，判断当前包是否是IPv4

```

version占4位，首部长度占4位,version = 4(IPv4), ipheader=20.
由于首部长度是以4字节为单位的-> version: 0100 ; 首部长度：0101

获取version:  0100 0101 & 0xFO(11110000) = 01000000 = 0x40
获取首部长度:  0100 0101 & 0x0F(00001111) = 0000 0101 = 5个4字节 = 20 Byte

```




### ping实现(c++&oc) 


#### 技术预研与构思 

根据ping的结果，我们需要解决以下问题： 

```
macdeiMac:PhoneNetSDK ethan$ ping www.baidu.com

PING www.a.shifen.com (61.135.169.121): 56 data bytes
64 bytes from 61.135.169.121: icmp_seq=0 ttl=49 time=32.559 ms
64 bytes from 61.135.169.121: icmp_seq=1 ttl=49 time=32.413 ms
64 bytes from 61.135.169.121: icmp_seq=2 ttl=49 time=32.489 ms
^C
--- www.a.shifen.com ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 32.413/32.487/32.559/0.060 ms
macdeiMac:PhoneNetSDK ethan$ 
```

* DNS解析(域名->ip) 
* 本地终端接收到的每个icmp包来自哪个主机
* icmp_seq
* ttl
* time 


以上问题解决方案如下： 

* DNS解析: socket支持
* 本地终端接收到的每个icmp包来自哪个主机: ip包中的source
* icmp_seq: icmp包中的 sequence number
* ttl:  ip包中的Time to live
* time: 发送包和接收到包时的时间差


#### 具体实现

IP包定义：
```
typedef struct PNetIPHeader {
    uint8_t versionAndHeaderLength;
    uint8_t differentiatedServices;
    uint16_t totalLength;
    uint16_t identification;
    uint16_t flagsAndFragmentOffset;
    uint8_t timeToLive;
    uint8_t protocol;
    uint16_t headerChecksum;
    uint8_t sourceAddress[4];
    uint8_t destinationAddress[4];
    // options...
    // data...
}PNetIPHeader;
```

ICMP包定义： 

```
/*
 use linux style . totals 64B
 */
typedef struct UICMPPacket
{
    uint8_t type;
    uint8_t code;
    uint16_t checksum;
    uint16_t identifier;
    uint16_t seq;
    char fills[56];  // data
}UICMPPacket;
```

构造ICMP包： 

```
+ (UICMPPacket *)constructPacketWithSeq:(uint16_t)seq andIdentifier:(uint16_t)identifier
{
    UICMPPacket *packet = (UICMPPacket *)malloc(sizeof(UICMPPacket));
    packet->type  = ENU_U_ICMPType_EchoRequest;
    packet->code = 0;
    packet->checksum = 0;
    packet->identifier = OSSwapHostToBigInt16(identifier);
    packet->seq = OSSwapHostToBigInt16(seq);
    memset(packet->fills, 65, 56);
    packet->checksum = [self in_cksumWithBuffer:packet andSize:sizeof(UICMPPacket)];
    return packet;
}
```

发送icmp包： 

```
 UICMPPacket *packet = [PhoneNetDiagnosisHelper constructPacketWithSeq:index andIdentifier:identifier];
        _sendDate = [NSDate date];
        ssize_t sent = sendto(socket_client, packet, sizeof(UICMPPacket), 0, (struct sockaddr *)&remote_addr, (socklen_t)sizeof(struct sockaddr));
        if (sent < 0) {
            log4cplus_warn("PhoneNetPing", "ping %s , send icmp packet error..\n",[self.host UTF8String]);
        }
```

接收icmp包： 

```
 size_t bytesRead = recvfrom(socket_client, buffer, 65535, 0, (struct sockaddr *)&ret_addr, &addrLen);
  if ((int)bytesRead < 0) {
            [self reporterPingResWithSorceIp:self.host ttl:0 timeMillSecond:0 seq:0 icmpId:0 dataSize:0 pingStatus:PhoneNetPingStatusDidTimeout];
            res = YES;
        }else if(bytesRead == 0){
            log4cplus_warn("PhoneNetPing", "ping %s , receive icmp packet error , bytesRead=0",[self.host UTF8String]);
        }else{
            
            if ([PhoneNetDiagnosisHelper isValidPingResponseWithBuffer:(char *)buffer len:(int)bytesRead]) {
                
                UICMPPacket *icmpPtr = (UICMPPacket *)[PhoneNetDiagnosisHelper icmpInpacket:(char *)buffer andLen:(int)bytesRead];
                
                int seq = OSSwapBigToHostInt16(icmpPtr->seq);
                
                NSTimeInterval duration = [[NSDate date] timeIntervalSinceDate:_sendDate];
                
                int ttl = ((PNetIPHeader *)buffer)->timeToLive;
                int size = (int)(bytesRead-sizeof(PNetIPHeader));
                NSString *sorceIp = self.host;
                
                
//                NSLog(@"PhoneNetPing, ping %@ , receive icmp packet..\n",self.host );
                [self reporterPingResWithSorceIp:sorceIp  ttl:ttl timeMillSecond:duration*1000 seq:seq icmpId:OSSwapBigToHostInt16(icmpPtr->identifier) dataSize:size pingStatus:PhoneNetPingStatusDidReceivePacket];
                res = YES;
            }
```

从接收到的buffer中分离icmp包： 

```
/* 从 ipv4 数据包中解析出icmp */
+ (char *)icmpInpacket:(char *)packet andLen:(int)len
{
    if (len < (sizeof(PNetIPHeader) + sizeof(UICMPPacket))) {
        return NULL;
    }
    const struct PNetIPHeader *ipPtr = (const PNetIPHeader *)packet;
    if ((ipPtr->versionAndHeaderLength & 0xF0) != 0x40 // IPv4
        ||
        ipPtr->protocol != 1) { //ICMP
        return NULL;
    }
    size_t ipHeaderLength = (ipPtr->versionAndHeaderLength & 0x0F) * sizeof(uint32_t);
    
    if (len < ipHeaderLength + sizeof(UICMPPacket)) {
        return NULL;
    }
    
    return (char *)packet + ipHeaderLength;
}
```


校验接收到的icmp包: 

```
+ (BOOL)isValidPingResponseWithBuffer:(char *)buffer len:(int)len
{
    UICMPPacket *icmpPtr = (UICMPPacket *)[self icmpInpacket:buffer andLen:len];
    if (icmpPtr == NULL) {
        return NO;
    }
    uint16_t receivedChecksum = icmpPtr->checksum;
    icmpPtr->checksum = 0;
    uint16_t calculatedChecksum = [self in_cksumWithBuffer:icmpPtr andSize:len-((char*)icmpPtr - buffer)];
    
    return receivedChecksum == calculatedChecksum &&
    icmpPtr->type == ENU_U_ICMPType_EchoReplay &&
    icmpPtr->code == 0 &&
    OSSwapBigToHostInt16(icmpPtr->identifier)>=KPingIcmpIdBeginNum;;
}
```

### TCP ping 

当有些服务器禁ping时，可以选择TCP ping。 

#### TCP ping原理

![](https://user-gold-cdn.xitu.io/2019/3/25/169b297949086692?w=554&h=199&f=jpeg&s=21655)


通过和目的主机及其端口建立TCP连接的方式计算其连接耗时。 


## traceroute 

### traceroute命令

```
/**** 设置每个路由发送的包数 ****/
traceroute -q 5 baidu.com

/**** 设置最大路由跳数 ****/
traceroute -m 5 baidu.com

/**** 不做DNS解析 ****/
traceroute -n baidu.com

/**** 绕过路由表直接发送到目的治具 ****/
traceroute -r baidu.com

/**** 使用ICMP包取代UDP包 ****/
traceroute -I baidu.com
```


### traceroute原理

tacceroute是利用增加存活时间(TTL)值来实现功能的。每当一个icmp包经过一个路由器时，其存活时间值就会减1，当其存活时间为0时，路由器便会取消包发送，并发送一个ICMP TTL封包给原封包发出者。

![](https://user-gold-cdn.xitu.io/2019/3/25/169b29794b0db1dc?w=779&h=406&f=jpeg&s=68496)


### traceroute过程

主叫方首先发出TTL = 1 的数据包，第一个路由器将 TTL 减1得0后就不再继续转发此数据包，而是返回一个ICMP超时报文，主叫方从超时报文中即可提取出数据包所经过的第一个路由器的地址。然后又发出一个TTL=2的ICMP数据包，可获得第二个路由器的地址，依次增加TTL便获取了沿途所有路由器位地址。

需要注意的是，并不是所有路由器都会如实返回ICMP超时报文。出于安全性考虑，大多数防火墙以及启动了防火墙功能的路由器缺省配置为不返回各种ICMP报文，其路由器或交换机也可被管理员主动修改配置变为不返回ICMP报文。因此Traceroute程序不一定能拿全所有沿途路由器地址。所以当某个TTL值的数据包得不到响应是，并不能停止这一追踪过程，程序仍然会把TTL递增而发出下一个数据包。一直达到预设或用于参数制定的追踪限制时才结束追踪。 

依据上述原理，利用了UDP数据包的Traceroute程序在数据包到达真正的目的主机时，就可能因为该主机没有提供UDP服务而简单将数据包丢弃，并不返回任何信息。为了解决这个问题，Traceroute故意使用了一个大于30000的端口号，因UDP协议规定端口号必须小于30000，所以目标主机收到数据包后唯一能做的事就是返回一个"端口不可达"的ICMP报文，于是主叫方就将端口不可达报文当做跟踪结束标志。

### 利用wireshark查看traceroute

我在命令行中traceroute www.baidu.com 以下是显示结果：

![image](https://user-gold-cdn.xitu.io/2019/3/4/169480089e6be093?w=1029&h=677&f=jpeg&s=243075)

如上图所示，UDP请求，第一个请求的端口是33435 ， 接下来的UDP请求，端口会递增。 

当到达目的地址时，目的地址会replay类型为3的包.

![image](https://user-gold-cdn.xitu.io/2019/3/4/169480089deb9ec2?w=1031&h=744&f=jpeg&s=264263)

如上图所示，是路由器返回的ICMP包，type是11。

### UDP traceroute的实现 

发送udp包，接收ip+icmp包,过滤route ip计算时间。

https://github.com/mediaios/net-diagnosis/tree/master/PhoneNetSDK/PhoneNetSDK/udptracert

### UDP traceroute存在的问题 

使用 UDP 的 traceroute，失败还是比较常见的。这常常是由于，在运营商的路由器上，UDP 与 ICMP 的待遇大不相同。为了利于 troubleshooting，ICMP 的request 和 replay 是不会封的，而 UDP 则不同。UDP 常被用来做网络攻击，因为 UDP 无需连接，因而没有任何状态约束它，比较方便攻击者伪造源 IP、伪造目的端口发送任意多的 UDP 包，长度自定义。所以运营商为安全考虑，对于 UDP 端口常常采用白名单 ACL，就是只有 ACL 允许的端口才可以通过，没有明确允许的则统统丢弃。比如允许 DNS/DHCP/SNMP 等。


### icmp traceroute 

发送icmp包，类型为8，每个路由返回的icmp包类型是11的超时包，当到达目的地址时，目的地址会replay类型为0的包

![](https://user-gold-cdn.xitu.io/2019/3/25/169b297962254ceb?w=765&h=356&f=jpeg&s=60153)


### icmp traceroute的实现 

https://github.com/mediaios/net-diagnosis/tree/master/PhoneNetSDK/PhoneNetSDK/utracert

## net-diagnosis(ios平台下网络诊断SDK)

`net-diagnosis`是ios平台下的网络诊断SDK，提供的功能有： 

* ping
* tcp ping
* traceroute
* icmp traceroute 
* nslookup 
* port scan 

项目地址： [github](https://github.com/mediaios/net-diagnosis)

后续更多关于网络诊断的功能会不断开发完善，欢迎提交[issue](https://github.com/mediaios/net-diagnosis/issues)

另，欢迎fork和star !

