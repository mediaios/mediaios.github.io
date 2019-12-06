# UNetState Service Support

## [阅读中文文档](https://mediaios.github.io/docs/unetstate/unetservice_ch)

## Introduction

For the students in the network and computer industry, you will always encounter some network-related problems. For example: You have accessed a service without success. At this time, you need to locate where the problem of the network request is. There are many reasons for a network request failure. For example, your device cannot access the external network, it may be a problem with DNS resolution of the domain name, or the network of the destination host may be down. Then you need a simple and lightweight tool to detect problems. That's why I developed this app. Let me briefly introduce.

`UNetState` currently provides 9 basic functions, which are: Ping, TCP Ping, UDP Traceroute, ICMP Traceroute, NSLookup, DNS, Port Scan, Scanning LAN Active Devices, Device Network Information. On the settings page, you can set parameters related to the above functions to more comprehensively troubleshoot problems, or you can send us some feedback on this page.

`UNetState` can be installed and used on the current mainstream Iphone and IPad. Therefore, whether you are a student, network engineer, programmer, or network operation and maintenance classmate, this app is very helpful for your study and work!

## Function Description

1. `Ping`:
The 'ping' command is mainly used to detect network connectivity and network transmission speed. It uses the ICMP protocol at the bottom. ICMP packets are contained in IP packets. Therefore, the ping command only goes to the IP layer. It can detect the connectivity of the lower three layers of TCP / IP. It does not have the concept of a port. You can set the number of ICMP packets sent in each ping on the app's settings page.

2. `TCP Ping`:
The 'ping' command can basically detect whether one host is connected to another host, but cannot detect whether it is connected to the application layer, and `TCP Ping` just solves this problem, it is a supplement to the ping command. You can set the number of packets and destination port for each TCP ping on the settings page.

3. `traceroute`:
This command is used to view the network routing path of network packets from your device to the destination host. In the implementation based on the UDP protocol, a port greater than 30,000 is usually selected for sending packets, but some host's firewall is in security or other considerations may prohibit UDP detection, so our implementation based on the ICMP protocol is for this situation A supplement. It can still try to find accurate routing information even if the traceroute packet is dropped by the server's firewall policy wall.

4. `NSLookup`:
Parsing domain name use local DNS. .

5. `DNS`:
That is, the user-defined DNS service. Google DNS service: 8.8.8.8 This function module is used by default. You can set up a third-party DNS service on the settings page to parse your domain name.

6. `Port Scan`:
This feature can list whether the ports commonly used by the service you want to probe are available. For example, http default port 80, https default port 443 and other commonly used ports.

7. `Scan for active devices in the LAN`
This function can quickly and accurately scan the IP of all active devices in the LAN. These devices include cell phones, computers, printers, and other networkable devices.

8. `Network Information`
This module can display the cellular network, WIFI network information of your mobile phone, and the external network information of your mobile phone.

 

## APP Overview

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_01.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_02.PNG" height="500px" alt="图片说明" > <img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_03.PNG" height="500px" alt="图片说明" >   
</div>

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_04.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_05.PNG" height="500px" alt="图片说明" > <img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_06.PNG" height="500px" alt="图片说明" >   
</div>

<div align="center">
<img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_07.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_08.PNG" height="500px" alt="图片说明" ><img src="https://raw.githubusercontent.com/mediaios/img_bed/master/UNet_09.PNG" height="500px" alt="图片说明" >   
</div>

## Others

If you have any better comments or suggestions for this app, you can send us an email through the 'Feedback' function on the settings page, or give us feedback by submitting an issue in the 'Open Source Library'. We will carefully evaluate and continuously update the app. Thank you! !! !!


Download it now and experience it!

