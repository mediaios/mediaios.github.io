# UNetState服务说明

## 说明 

对于从事网络和计算机行业的同学来说，你总会遇到一些网络相关的问题。比如：你访问了一个服务没有成功，此时就需要你定位到这次网络请求的问题出现在哪一环节。网络请求失败的原因有很多，比如可能是你的设备本身访问不了外网、可能是进行域名DNS解析出了问题、也可能是目的主机的网络不通。那么就需要一款简单轻量级的工具检测问题。这就是我开发这款应用的原因。下面我来简单介绍一下。

'UNetState'目前提供了9大基本功能，它们分别是：Ping、TCP Ping、UDP Traceroute、ICMP Traceroute、NSLookup、DNS、Port Scan、扫描局域网活跃设备、设备网络信息。在设置页面，你可以设置和上述功能相关的参数更全面的排查问题，也可以在该页面给我们发一些意见反馈。

'UNetState'可在目前主流的Iphone以及IPad上安装使用。因此，不管你是学生、网络工程师、程序员、还是网络运维同学，这款应用都对你的学习和工作非常有帮助！！！

## 功能描述 

1. `Ping`: `ping`命令主要用来检测网络连通性以及网络传输速度，它底层采用的是ICMP协议，ICMP包是被包含在IP包内的。因此ping命令只到IP层，它能很好的检测到TCP/IP的下三层连通性，它没有端口的概念。你可以在应用的设置页面设置每次ping时发送ICMP包的数量。
2. `TCP Ping`: `ping`命令基本上能检测一台主机到另一台主机是否连通，但是无法检测到对于应用层是否连通，而`TCP Ping`正好解决了这一问题，它是ping命令的一个补充。你可以在设置页面设置每次`TCP ping`时发包数量以及目的端口。
3. `traceroute`: 该命令用于查看网络数据包从你的设备到目的主机的网络路由路径。在基于UDP协议的实现中通常是选择一个大于30000的端口做发包，但是有些主机的防火墙处于安全或其它方面的考虑可能会禁止UDP的探测，所以我们基于ICMP协议的实现是对这种情况的一个补充。它可以在traceroute数据包被服务器的防火墙策略墙丢弃的情况下仍然能尝试着查到精确的路由信息。
4. `NSLookup`: 利用本地DNS做域名解析。
5. `DNS`: 即用户自定义DNS服务。谷歌DNS服务：8.8.8.8 在该功能模块是被默认用的。你可以在设置页面设置第三方DNS服务去解析你的域名。
6. `Port Scan`: 该功能可以列出你要探测的服务常用的端口是否可用。比如http默认端口80、https默认端口443等常用端口。
7. `扫描局域网内的活跃设备`: 该功能能可以快速准确地扫描局域网中所有活动设备的IP。 这些设备包括手机，计算机，打印机和其它可联网设备。
8. `网络信息`:该模块可以展示你手机的蜂窝网络、WIFI网络信息以及手机的外网信息等。

## 应用概览 

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_01.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_02.PNG" height="500px" alt="图片说明" > <img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_03.PNG" height="500px" alt="图片说明" >   
</div>

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_04.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_05.PNG" height="500px" alt="图片说明" > <img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_06.PNG" height="500px" alt="图片说明" >   
</div>

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_07.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_08.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_09.PNG" height="500px" alt="图片说明" >   
</div>

## 其它

如果你对该款应用有什么更好地意见或建议，你可以通过设置页面的'意见反馈'功能给我们发邮件，或者通过在'开源库'中提交issue的方式给我们反馈。我们会认真评估并持续更新应用。谢谢！！！

现在赶快下载体验吧!!!