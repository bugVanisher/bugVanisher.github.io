---
layout: post
title: 架构与核心类——locust源码分析
categories: [源码分析]
description: 介绍了locust的架构及其核心类
keywords: locust, 架构, 核心类
---

这段时间在研读压力测试框架locust的源码，我打算分三篇文章好好剖析一下这个开源框架，一起体验编程乐趣吧~

我将会从三个方面分析locust的源码，分别是架构和核心类、执行器状态机和通信机制、第三方库实现。这是第一篇，架构与核心类。

在进入locust的源码分析之前，我们先来简单看看目前的一些压测工具：

