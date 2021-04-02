---
layout: post
title: 我是如何构建博客的——方法和工具
categories: [生活]
description: 分享我是如何使用现有的工具和平台创建和维护自己的博客
keywords: 搭建, 构建, 博客, 工具
published: false
---

写博客也有一段时间了，逐渐形成了自己的习惯和工具栈，今天分享出来，供有需要的朋友参考参考。条条大路通罗马，找到适合自己的，然后坚持下去！

首先需要说明的是，以下方案搭建的博客是完全由自己控制的，从页面的样式，到写文章再到发布，随之而来的可能是繁琐的重复工作，然而获得了极大的自由度，毕竟发文章并不是非常高频的操作。本文主要为了介绍构建博客过程中遇到的主要问题以及解决方法，因此并不会非常详细的介绍每一个步骤，具体的可以自行google。

# 第一部分——搭建

## 1、静态站点生成器

我们写文章的时候是使用markdown语法的，md可以让我们专注于内容本身，而不需要关心其他。静态站点生成器负责将我们写的md格式文章转化为固定格式的静态文件，只需要放到一个简单的web服务器上就可以查看了。[jekyll](https://jekyllrb.com/) 就是这样一种生成器，它是基于ruby实现的，所以需要ruby环境，其实类似的工具有很多，之所以选择它，因为它由github官方支持，我们的博客最终需要部署到github pages。

## 2、模板选择

如果开发一个静态站点，需要做哪些内容？相信有经验的朋友能轻易说出来，比如需要设计页面，写css，分页，写js实现一些动态效果等等。开玩笑吧，我就写文章而已，还搞这些？所以，模板出现了。[jekyll模板](http://jekyllthemes.org/) ，选择一个自己喜欢的模板，然后开始吧！

## 3、GitHub Pages

假设我们已经把模板下载，使用Jekyll生成了静态文件了，我们的文件放哪里访问呢？当然是web服务器啦，额，没有服务器，又不想花钱买服务器怎么办，那就用github pages吧。[GitHub Pages](https://pages.github.com/) 配置好后，默认为我们生成一个类似bugvanisher.github.io的域名，前面的bugvanisher是我github的账号名。

## 4、域名申请

看到bugvanisher.github.io这样的域名，我们可能不太满意了，一点都不个性，太github了。我们可以在阿里云或者腾讯云买一个域名，这里我以阿里云为例。

域名生效后，添加一条CNAME记录到bugvanisher.github.io，如下：

![image-20210402174455137]({{ site.cdn.gh-images }}/static/image-20210402174455137.png)

然后在github对应的博客仓库根目录中，添加CNAME文件，内容为bugvanisher.cn。

## 5、静态资源加速

国内访问github确实会比较慢，所以在打开页面的时候加载很慢，jsDelivr可以将静态资源，如js，css，图片等资源缓存到cdn中，因此可以加快页面的加载速度。这个需要配合模板，统一将静态资源的引用从自站改到jsDelivr。



# 第二部分——写

## 1、MarkDown编辑器

支持md的编辑器有很多了，我喜欢用[Typora](https://typora.io/)，因为所写即所见。此外，他还能配合图片上传工具，如果写文章的时候需要插入本地图片，只需要复制粘贴，然后右键上传，即可上传到指定的位置，并返回url地址。

类似这样：

```shell
![image-20210402174455137]({{ site.cdn.gh-images }}/static/image-20210402174455137.png)
```

这样真的免去了很多图片上传的工作。

## 2、图片上传

上面提到在编辑器中上传图片，这里其实是用到了[picgo](https://github.com/Molunerfinn/PicGo), 我们在Typora中设置好用picgo作为图片上传工具，如下：

![image-20210402181201887]({{ site.cdn.gh-images }}/static/image-20210402181201887.png)

在picgo中设置好将图片上传到github的图床（自己创建一个public仓库作为图床），当我们在编辑器中上传图片时，实际上是将图片上传到我们的图床仓库，配合jsDelivr，图片资源的访问也会加快。

<img src="{{ site.cdn.gh-images }}/static/image-20210402181521755.png" alt="image-20210402181521755" style="zoom:50%;" />

## 3、文章来源分类

这一个功能是我自己 [实现](https://github.com/bugVanisher/bugVanisher.github.io/commit/af59a4388bb7a3e7ed40478b47c1cb879f939a50) 的，有需要可以参考。目前我把我的博客文章分为三类，分别是原创、转载、翻译，配合模板，我只需要在我对应博客文章的md文件添加：

```
type: reproduce
or
type: translate
```

这样出来的效果如下：

<img src="{{ site.cdn.gh-images }}/static/image-20210402182151951.png" alt="image-20210402182151951" style="zoom:50%;" />



# 第三部分——推广

## 1、Google收录

写了文章当然想分享，让更多人看到，所以我们需要做SEO，做搜索引擎收录，想让谷歌收录基于github pages的博客非常简单，在 Google Search Console 提交自己的网址即可。jekyll模板一般会集成sitemap生成插件的，我们把sitemap地址提交给google即可。

![image-20210402184557692]({{ site.cdn.gh-images }}/static/image-20210402184557692.png)

## 2、百度收录

Github Pages是拒绝百度搜索引擎爬虫的，所以想让百度收录，只能部署一个镜像博客。可以参考这篇文章中的方案：[如何让百度收录 GitHub Pages 个人博客](https://zhuanlan.zhihu.com/p/111773896) ，文中最后提到了两个方案，一个是替换阿里云的DNS地址，一个是创建一个A记录到镜像博客IP。文中作者选择了第一种，而我选择了第二种，而且只针对百度爬虫，其他流量还是走的github pages而不是镜像服务。

<img src="{{ site.cdn.gh-images }}/static/image-20210402184459014.png" alt="image-20210402184459014" style="zoom: 67%;" />



