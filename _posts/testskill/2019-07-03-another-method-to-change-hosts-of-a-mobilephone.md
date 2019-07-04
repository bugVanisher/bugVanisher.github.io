---
layout: post
title: "另一种'修改'手机hosts的方法"
author: "见欢"
categories: 测试技巧
description: 
keywords: hosts,代理,抓包
---

<a name="xUpb6"></a>
## 前言
在某些场景下我们需要修改手机的hosts，比如指定访问预发服务器，一个常见修改hosts的方法是：对已root手机，修改/etc/hosts文件。然而出于手机安全的考虑，越来越多的手机厂商升级反root机制，手机root的难度越来越大，加上修改手机上的hosts本身也比较麻烦，我们迫切需要另外一种方法来修改手机的hosts。

我们想到，通过代理来绕过手机本地hosts，通过修改代理服务器（后面简称proxy）的hosts，间接地改变手机的hosts，这样我们就能让手机访问指定的IP。随之而来的问题是，https的代理有时候并不能正常使用，也就是将proxy的证书装进手机也不一定能够正常代理（the client failed to negotiate an ssl connection）。这种代理方式其实是对https抓包的，经过学习了解，其实可以采用http隧道的方式进行代理，过程并没有对https的包进行抓取，proxy只是代替手机和真实服务器建立安全的TCP连接（应用层TLS），而请求、响应报文对代理服务器而言是加密的，这就好比一个不知道快递内容的快递员。

为了让proxy与真实服务器建立连接，客户端需要主动使用“CONNECT”方法的http请求（包含服务器地址和端口）给proxy，proxy收到请求后与真实服务器建立安全连接。如下图

![image.png]({{ site.url }}/assets/testskill/http-connect.png)<br />

需要注意的是“CONNECT”方法只有在客户端使用代理服务器的时候才会使用到。

![image.png]({{ site.url }}/assets/testskill/wireshark数据包.png)<br />

上图展示了客户端与proxy建立tls连接的过程，在proxy返回“200 Connection established”后，实际上proxy已经与真实服务器建立了连接。往后客户端和真实服务器的数据就通过proxy进行传输，而proxy无法看到数据包的真实内容。

基于上述理论，设置代理服务器，不进行https代理，而是使用http-tunnel，即可快速让手机走代理服务器网络（通常是我们自己的pc电脑），指定hosts。下面分别讲述两个工具的设置http隧道的方法。

<a name="bwhs4"></a>
## 设置Charles
软件的介绍和安装这里不做赘述了，网上太多了，这里只讲设置方法。<br />
<br />1）Proxy-Proxy Settings，勾选“Enable transparent HTTP proxying”<br />
<br />![image.png]({{ site.url }}/assets/testskill/Charles-proxy.png)<br />
<br />2）Proxy-SSL Proxy Settings，不勾选“Enable SSL Proxying”

![image.png]({{ site.url }}/assets/testskill/Charles-ssl-proxy.png)<br />

<a name="HadGN"></a>
## 设置Burpsuite
1）Proxy-Option-SSL Pass Through勾选“**automatically add entries on client SSL negotiation failure**”<br />![image.png]({{ site.url }}/assets/testskill/burp-ssl-pass.png)<br />  当然如果可以在手机安装代理服务器的证书，直接抓取https的包是最好的（网上已有很多教程，这里也不赘述了）。实际情况是，proxy并不能对所有的https域名完成安全连接的建立，从而顺利抓到包的，所以抓不到的就用http-tunnel吧。

<a name="vgWgc"></a>
## 参考链接
[https://www.joji.me/en-us/blog/the-http-connect-tunnel/](https://www.joji.me/en-us/blog/the-http-connect-tunnel/)

[https://blog.csdn.net/bohu83/article/details/78437982](https://blog.csdn.net/bohu83/article/details/78437982)

[https://www.charlesproxy.com/](https://www.charlesproxy.com/)

[https://portswigger.net/](https://portswigger.net/)


