---
layout: post
title: 直播推流sdk测试简介
categories: [直播]
description: 介绍直播推流SDK相关测试内容和方法
keywords: 直播SDK, 推流, 测试
published: false
---

## 前言
2019年底，我有幸加入直播相关的业务团队，开始负责直播SDK和CDN转推相关的测试工作。其中我们直播SDK基本从0做起，完成从推流端到拉流端实现，所以我将会用直播系列文章记录整个的测试过程，当然，主要是以测试的角度。

直播SDK包含推流和拉流，这两块测试内容我将会分两篇文章讲述，这一篇讲的是——推流SDK。

在讲直播SDK测试之前，我们先来了解一下整个的直播业务架构从而更好地理解我们究竟要测试什么内容。
![](https://user-gold-cdn.xitu.io/2018/3/26/162601f0836f2cb7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 测试内容
...