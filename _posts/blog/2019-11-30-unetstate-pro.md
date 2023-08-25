---
layout: post
title: UNetState Pro technical service support
description: APP Store Application UNetsTate Pro Technical Service Support
category: blog
tag: UNetState Pro, ping,tracert,traceroute,DNS,lan scan,netinfo,局域网扫描
---

## UNetState Pro Introduce

UNetState Pro is a network detection tool for mobile devices. Including obtaining basic network information of mobile phone, route tracking, ping, TCP connection, DNS resolution information, LAN device scanning, etc. Simple operation and clear interface, it is a very convenient and easy-to-use network tool.

Function details： 

1. Ping:
ping (Packet Internet Groper), an Internet packet explorer, a program for testing network connectivity. Ping sends an ICMP; echo request message to the destination and reports whether the desired ICMP echo (ICMP echo response) is received. It is a command used to check whether the network is unblocked or the speed of the network connection. 

2. TCP Ping: 
Many times, we need to test the TCP port. Although the ping command works well, it cannot test the port. Because ping is based on the ICMP protocol and belongs to the IP layer protocol, it cannot test the TCP / UDP port of the transport layer. For TCP PING, it is a supplement to Ping. We use it to test whether the host port is connected when the Ping fails.

3. UDP Traceroute: 
Through traceroute we can know the routing path of network packets from your computer to the host at the other end.
In the UDP-based implementation, the data packets sent by the client are transmitted through the UDP protocol, using a port number greater than 30,000. When the server receives this data packet, it will return an ICMP error message that the port is unreachable. The client determines whether the data packet reaches the target host by judging whether the received error message is a TTL timeout or the port is unreachable.

4. ICMP Traceroute:
UDP traceroute is invalid in some cases. For example, if the server processes UDP packets, the traceroute using UDP protocol will fail. So we use ICMP packets instead of UDP packets to send in IP packets, and analyze the ICMP packets and other related values in the returned IP data packets. This prevents our traceroute packets from being dropped by the server's firewall policy wall

5. lookup :
Parsing domain name use local DNS. 

6.DNS(Custom DNS):
Use third-party DNS servers parse domain name. Google DNS service 8.8.8.8 is added by default in the app. You can also add other DNS services in the settings.

7. LAN Scan: 
Quickly and accurately scan the IP of all active devices in the local area network. These devices include mobile phones, computers, printers and other networkable devices.

8. Network info: 
Display local&cellular &public network information.  

## APP Information

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_01.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_02.PNG" height="500px" alt="图片说明" > <img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_03.PNG" height="500px" alt="图片说明" >   
</div>

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_04.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_05.PNG" height="500px" alt="图片说明" > <img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_06.PNG" height="500px" alt="图片说明" >   
</div>

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_07.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_08.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNetPro_09.PNG" height="500px" alt="图片说明" >   
</div>

