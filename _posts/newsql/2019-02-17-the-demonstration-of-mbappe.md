---
layout: post
title: "测试阶段慢查询分析系统——实现"
author: "见欢"
categories: 慢查询分析系统
description: 测试阶段慢查询分析系统，我们是如何实现的。
keywords: 慢查询, 实现
---

### 一、技术栈
拥抱开源技术， mbappe 慢查询分析系统使用开源软件实现，以下是各个应用使用的技术栈：
##### agent
* jvm-sanbox
* rocketmq

##### worker
* rocketmq
* spring-boot

##### web
* spring-boot
* vue2.x
* webpack
* element-UI

### 二、效果展示
下面将会展示 mbappe 系统，敏感信息已做处理。

##### 1、sql 列表
![]({{ site.url }}/assets/newsql/show/整体.png) 

此处默认展示近7天的，未处理、待确认、跟进中的 sql 列表
不展示已处理、已忽略、误报的sql

##### 2、配置通知
在配置通知页面配置订阅人，则系统会将可疑的 sql 通知到订阅人（目前只有钉钉通知），配置如下
![]({{ site.url }}/assets/newsql/show/告警配置.png) 

当系统分析 sql 时，发现严重或警告等级的 sql 时，会立即（默认，暂时不支持定时推送）通知到订阅人。
![]({{ site.url }}/assets/newsql/show/钉钉通知.png) 

##### 3、sql 详情
点击查看，进入到 sql 详情页，将会看到 sql 的具体情况，包含 参数化 sql 、原 sql 、原始查询计划结果，分析结果。
![]({{ site.url }}/assets/newsql/show/详情.png) 
详情页展示了系统初步分析的所有结果，如下例子：
![]({{ site.url }}/assets/newsql/show/explain结果.png) 


##### 4、查看堆栈
点击查看堆栈，可以看到产生这条 sql 的整个调用栈，方便找到出处。
![]({{ site.url }}/assets/newsql/show/堆栈.png) 

##### 5、查看表结构
立即查看表结构，方便定位
![]({{ site.url }}/assets/newsql/show/表结构.png) 

##### 6、处理 sql
如果一条 sql  正在跟进，那么我们就需要处理它，点击处理，输入问题描述和解决方式。
![]({{ site.url }}/assets/newsql/show/处理对话框.png) 

##### 7、查看操作记录
![]({{ site.url }}/assets/newsql/show/操作记录.png) 

以上便是 mbappe 系统的基本功能，一起期待更多更好玩的功能吧！