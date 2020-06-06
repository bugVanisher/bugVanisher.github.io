---
layout: post
title: 一个弱网环境搭建的简单方案
categories: [测试技巧]
description: 使用mac Network Link Conditioner 实现弱网环境搭建
keywords: 弱网环境, 弱网搭建
published: true
---

> 很多时候，我们需要对应用进行弱网环境下的测试，验证应用在网络较差的情况下的表现。实现弱网环境搭建的方案有很多，本文介绍一种相对比较简单但非常好用的方案。

说到弱网环境搭建，稍微搜索一下就能找到以下几类方案：

* 基于Charles、Fiddler一类的代理软件
* 使用FaceBook 的ATC方案
* 购买专业的网络设备，构建大范围的弱网环境
* 手机直接使用Qnet相关弱网模拟软件
* 使用PC安装弱网模拟软件，共享网络给移动设备，实现弱网模拟

以上方案都有各自的优势和劣势，综合考虑后不难发现，最后一种方案是最好的，因为它不仅相对简单，还能实现多终端、跨系统接入，并且抓包。在Windows系统下有clumsy、netlimite这类软件，而本文介绍的是Mac下的实现方式。

Network Link Conditioner，这是苹果提供的弱网模拟工具，支持iOS和Mac系统。下面分几个步骤完成弱网环境的搭建：

## 一、Mac安装Network Link Conditioner
1、点击 [下载地址](https://developer.apple.com/download/more/) 访问苹果开发者网站提供的下载页面，NLC这个工具被包含在Hardware IO Tools for Xcode的工具包中。

![]({{ site.url }}/assets/testskill/nlc_download.jpg)

2、双击下载后的安装包，找到NLC双击安装

![]({{ site.url }}/assets/testskill/nlc_install.jpg)

## 二、Mac设置为有线网络联网
公司一般提供两种上网方式，分别是有线和无线，这里我们的Mac使用有线上网（需要一个有线网卡，本方案使用的是绿联的转换器，集成了有线网卡），Mac自带的无线网卡用来共享网络。
![]({{ site.url }}/assets/testskill/cable_network.jpg)

## 三、设置共享网络
在Mac 系统偏好设置——共享 中 选中互联网共享, 共享以下来源的连接，选中有线网络，用以下端口共享给电脑中选中WiFi。

![]({{ site.url }}/assets/testskill/share_network.jpg)

点击WiFi选项，设置共享网络名称和密码，这里我设置的网络标识为:bugVanisher。

![]({{ site.url }}/assets/testskill/wifi_options.jpg)

勾选中互联网共享，并开启网络共享,出现以下图片的情况说明共享成功。

![]({{ site.url }}/assets/testskill/share_inprocess.jpg)

## 四、配置弱网
此时使用手机连上bugVanisher,Mac上打开Network Link Conditioner,这里需要注意Network Link Conditioner是一个配置面板，所以不能在安装的应用中找到它，我们可以使用聚焦搜索来找到他并打开。

![]({{ site.url }}/assets/testskill/nlc_pane.jpg)

接下来就可以根据需要配置各种网络情况了，比如带宽、丢包、延时等。关于如何设置的问题，网上有很多了，这里不再赘述。



## 五、遇到的问题

1、如果在开启共享后，手机搜索不到对应的WIFI信号，则可能是与附近的频段冲突了，修改为其他的频段试试。

<img src="https://bugvanisher.github.io/images/static/image-20200528171332231.png" alt="image-20200528171332231" style="zoom:50%;" />

2、共享后WIFI模块可能通过有线网自动获取IP地址，这并不是我们想要的，此时手动设置IP地址即可，路由器填写有线网对应的路由器，类似按如下配置：

<img src="https://bugvanisher.github.io/images/static/image-20200528171856392.png" alt="image-20200528171856392" style="zoom:50%;" />

## 小结
这是一套相对简单的弱网模拟环境的搭建方案，最大的好处除了可以多终端接入，使用wireshark抓包也非常方便，对于协议分析非常友好。实际上本文完全也可以作为一个抓包分析环境搭建的案例。
>参考连接:
>
>https://www.jianshu.com/p/fb4824fd5bbc
>
>https://www.520mwx.com/view/53152
>
>https://juejin.im/post/5c739cbbf265da2dd773e944