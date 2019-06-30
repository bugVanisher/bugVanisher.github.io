---
layout: post
title: 如何给安卓apk包签名
categories: [Android]
description: 安卓apk的签名方式和过程
keywords: 数字证书,签名,打包
---
<a name="wyt9qi"></a>
## [](#wyt9qi)一、非混淆打包流程
![]({{ site.url }}/assets/android/20190515/非混淆打包.png)

混淆方式的签名，后面再单独来讲。

<a name="ilu9sf"></a>
## [](#ilu9sf)二、安卓apk包结构
APK就是一个zip压缩包，解开这个APK包我们可以看到以下的结构.<br />![]({{ site.url }}/assets/android/20190515/apk包结构.png)


<a name="v6a1cu"></a>
## [](#v6a1cu)三、app加固
<a name="plvwul"></a>
### [](#plvwul)1、为何加固

- 防篡改


通过完整性保护和签名校验保护，能有效避免应用被二次打包，杜绝盗版应用的产生。

- 防逆向


对代码进行隐藏及加密处理，使攻击者无法对二进制代码进行反编译，获得源代码或代码运行逻辑。

- 防调试


通过反调试技术，使攻击者无法调试原生代码或Java代码，阻止攻击者获取代码里的敏感数据。

- 数据保护


提供安全键盘、通讯协议加密、数据存储加密、异常进程动态跟踪等功能技术，在各个环节有效阻止数据被捕获、劫持和篡改
<a name="n1hgxe"></a>
### [](#n1hgxe)2、第三方加固

- **阿里聚安全** 链接：[jaq.alibaba.com/](http://jaq.alibaba.com/),已下线

- 腾讯云应用乐固 链接：[https://cloud.tencent.com/product/ms?idx=2](https://cloud.tencent.com/product/ms?idx=2)

- 360加固保 链接：[jiagu.360.cn/](http://jiagu.360.cn/)

- 梆梆加固 链接：[dev.bangcle.com/](http://dev.bangcle.com/)

- 爱加密 链接：[safe.ijiami.cn/](http://safe.ijiami.cn/)


<a name="l72qai"></a>
## [](#l72qai)四、app签名
<a name="b84fge"></a>
### [](#b84fge)1、数字签名and数字证书
  数字证书是采用数字手段来证实用户身份的一种方法。数字证书含有两部分数据：一部分是对应主体（单位或个人）的信息，另一部分是这个主体所对应的公钥。即数字证书保存了主体和它的公钥的一一对应关系，用于自我认证（向其他的用户证明自己的身份）。
  
  Java中的keytool可以用来创建数字证书，所有的数字证书是以一条一条(采用别名区别)的形式存入证书库的中，证书库中的一条证书包含该条证书的私钥，公钥和对应的数字证书的信息。证书库中的一条证书可以导出数字证书文件，数字证书文件只包括主体信息和对应的公钥。

<a name="y8i5ni"></a>
#### [](#y8i5ni)                数字签名的位置和作用
![]({{ site.url }}/assets/android/20190515/密码安全学.png)<br />  
<a name="6a70bz"></a>
### [](#6a70bz)2、为什么要签名
  Android系统要求每一个安装进系统的应用程序都是经过数字证书签名的，数字证书的私钥则保存在程序开发者的手中。Android将数字证书用来标识应用程序的作者和在应用程序之间建立信任关系，而不是用来决定最终用户可以安装哪些应用程序。这个数字证书并不需要权威的数字证书签名机构认证，它只是用来让应用程序包自我认证的。
<a name="7o73ch"></a>
#### [](#7o73ch)应用程序升级
  如果希望用户无缝升级到新的版本，那么必须用同一个证书进行签名。这是由于只有以同一个证书签名，系统才会允许安装升级的应用程序。如果采用了不同的证书，那么系统会要求应用程序采用不同的包名称，在这种情况下相当于安装了一个全新的应用程序。如果想升级应用程序，签名证书要相同，包名称要相同。
<a name="mt0vra"></a>
#### [](#mt0vra)应用程序模块化
  Android系统可以允许同一个证书签名的多个应用程序在一个进程里运行，系统实际把他们作为一个单个的应用程序，此时就可以把我们的应用程序以模块的方式进行部署，而用户可以独立的升级其中的一个模块。
<a name="r128qk"></a>
#### [](#r128qk)代码或者数据共享
  Android提供了基于签名的权限机制，那么一个应用程序就可以为另一个以相同证书签名的应用程序公开自己的功能。以同一个证书对多个应用程序进行签名，利用基于签名的权限检查，你就可以在应用程序间以安全的方式共享代码和数据了。不同的应用程序之间，想共享数据，或者共享代码，那么要让他们运行在同一个进程中，而且要让他们用相同的证书签名。
<a name="nunrsz"></a>
### [](#nunrsz)3、如何签名
![]({{ site.url }}/assets/android/20190515/如何签名.png)
<br />app签名机制主要依靠META-INF目录里的三个文件：
```
MANIFEST.MF
CERT.RSA
CERT.SF
```

<br />这三个文件的相互关系如下：<br />![]({{ site.url }}/assets/android/20190515/meta.png)<br />用户从市场下载APK安装文件，在真正安装APK前，会首先验证数字签名。具体步骤：<br />
<br />(a) 首先计算除META-INF文件夹以外所有文件的SHA1摘要值，同META-INF文件夹内的MANIFEST.MF文件中的摘要值做比对。如果不同，则证明源文件被篡改，验证不通过，拒绝安装。<br />
<br />(b) 计算META-INF文件夹中的MANIFEST.MF的摘要值， 以及MANIFEST.MF中每一个摘要项的摘要值，同.SF文件中的摘要值做比对。如果不同，则证明.SF被篡改，验证不通过，拒绝安装。<br />
<br />从META-INF文件夹中的.RSA 文件中取出开发者证书，然后从证书中提取开发者公钥，用该公钥对.SF文件做数字签名，并将结果同.RSA文件中的.SF签名进行比对。如果不同，则验证不通过，拒绝安装。

<a name="7ruogb"></a>
#### [](#7ruogb)小结：

1、数据指纹，签名文件，证书文件的含义<br />	
* a、数据指纹就是对一个数据源做SHA/MD5算法，这是唯一值<br />	
* b、签名文件技术就是：数字签名+RSA算法+SHA摘要<br />	c、证书文件中包含了公钥信息和其他信息<br />
* d、在Android签名之后，其中SF就是签名文件，CERT.RSA就是证书文件我们可以使用openssl或keytool查看RSA文件中的证书信息和公钥信息<br />
* e、同一个证书可以对多个应用签名<br />

2、Android中的签名有两种方式：jarsigner和signapk 这两种方式的区别是：<br />	
* a、jarsigner签名时，需要的是keystore文件，而signapk签名的时候是pk8,x509.pem文件<br />	
* b、jarsigner签名之后的SF和RSA文件名默认是keystore的别名，而signapk签名之后文件名是固定的:CERT<br />


```bash
#查看方式
#keystoe的rsa文件
keytool -printcert -file CERT.RSA

#x509格式证书
openssl pkcs7 -inform DER -in CERT.RSA -print_certs
```
* c、Eclipse、Idea中我们在跑Debug程序的时候，默认用的是jarsigner方式签名的，用的也是系统默认的debug.keystore签名文件<br />
* d、keystore文件和pk8,x509.pem文件之间可以互相转化

<a name="tdwvtv"></a>
#### [](#tdwvtv)敲黑板：
(a) Android签名机制其实是对APK包完整性和发布机构唯一性的一种校验机制。

(b) Android签名机制不能阻止APK包被修改（二次打包），但修改后的再签名无法与原先的签名保持一致。

(c) APK包加密的公钥就打包在APK包内，且不同的私钥对应不同的公钥。换言之，不同的私钥签名的APK公钥也必不相同。所以我们可以根据公钥的对比，来判断私钥是否一致。

<a name="eexrfn"></a>
### [](#eexrfn)4、证书管理
  一般自己生成证书，并自己管理密钥和密码库<br />

<a name="mgwekw"></a>
## [](#mgwekw)五、参考文档
什么是数字签名<br />[http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)<br />

常见加固内容<br />[https://cloud.tencent.com/document/product/283/13769](https://cloud.tencent.com/document/product/283/13769)<br />

发布安卓应用<br /> [https://developer.android.com/studio/publish/?hl=zh-cn](https://developer.android.com/studio/publish/?hl=zh-cn)<br />

安卓签名机制<br />[http://docs.ioin.in/writeup/www.arkteam.net/fa860d69-16c4-4ed1-8fd7-06fbab513d97/index.html](http://docs.ioin.in/writeup/www.arkteam.net/fa860d69-16c4-4ed1-8fd7-06fbab513d97/index.html)
