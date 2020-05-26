---
layout: post
title: 一些我们需要知道的网络抓包工具
categories: [网络]
description: 介绍网络抓包的常用工具
keywords: 抓包, 代理
published: true
---

这篇文章介绍一些我认为比较好用，常用到的抓包工具，具体用法网上有很多，了我就不赘述了，这里主要讨论它们的区别与使用场景。我的观点是，没有最好的工具，只有最合适的工具。

## Http代理抓包

以下是http代理抓包工具：

* Fiddler 是 freeware， 以前只有Windows版，最近它也出了 macOS 版本。
* Charles 是商业软件，之前只有macOS版本，现在它也有 Windows 版本了，由于使用 JAVA 编写，它的 Windows 版很容易实现。
* Burp Suite 基于Java编写，是用于攻击web 应用程序的集成平台。它包含了许多工具，并为这些工具设计了许多接口，以促进加快攻击应用程序的过程。
* Whistle 是基于 Node 实现的跨平台抓包调试代理工具。
* Mitmproxy 免费的、开源的交互式的http代理工具，提供UI和命令行模式。

|工具名称 |	编写语言 | 跨平台	|	是否支持扩展 | websocket代理 | 是否开源|
| :--- | ---- | ---- | ---- | ---- | ---- |
|Fiddler | .NET Framework |  原生Windows，已支持MacOS | 支持 | 支持查看,但修改麻烦 | 是|
|Charles | Java |  原生MacOS，有Windows版 | 不支持 | 不支持 | 商业软件|
|Burp Suite | Java |  完全跨平台 | 支持 | 支持查看 | 分专业版、免费版|
|Whistle | Node |  完全跨平台 | 支持 | 支持且较全面 | 是|
|Mitmproxy | Python、C |  完全跨平台 | 支持 | 不支持 | 是|



## TCP层抓包

除了http协议的抓包工具，还有两个需要掌握的TCP层抓包工具：

* tcpdump 是一款sniffer工具，它可以打印所有经过网络接口的数据包的头信息，也可以使用-w选项将数据包保存到文件中，方便以后分析。

* wireshark 最重要的且被广泛使用的网络协议分析工具。主要功能是撷取网络封包，并尽可能显示出最为详细的网络封包资料。

tcpdump是一个命令行工具，很多时候结合wireshark（可视化工具）一起使用，先用tcpdump将网络数据包抓取并导出为.pcap文件，然后使用wireshark进行分析。

## 真机抓包
有些应用比较皮，不走操作系统的 HTTP 代理。还有些应用直接走 TCP 协议，无法使用 HTTP 代理抓包。

iOS 5后，Apple引入了RVI remote virtual interface的特性，它只需要将iOS设备使用USB数据线连接到mac上，然后使用rvictl工具，以iOS设备的UDID为参数（可用iTunes查看）在Mac中建立一个虚拟网络接口rvi，就可以在mac设备上使用tcpdump，wireshark等工具对创建的接口进行抓包分析。

Android实际上是linux系统，和PC操作系统一样可以使用tcpdump来抓包，但是需要root权限。

PacketCapture 和 tPacketCapture 可以直接安装在 Android 设备上，他们会在设备上启动一个 VPN，让所有的网络流量都从 VPN 经过，从而实现抓包。这两个 App 在安装的时候都不需要 Root 权限。

## 更简单的方法
桌面系统上的 Wireshark 和 tcpdump 可以对该系统的网络设备抓包。因此我们只需保证移动设备的所有流量经过这个设备的某个网卡，就能得到它传输的所有信息。

把桌面设备变成一个热点，让移动设备通过该热点上网就能做到这点了。无论桌面设备使用有线网络还是无线网络，只需要购买一个无线网卡就能实现热点功能。不用做任何配置，就能抓包了。

