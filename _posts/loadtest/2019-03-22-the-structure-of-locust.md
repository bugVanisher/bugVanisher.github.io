---
layout: post
title: 架构与核心类——locust源码分析
categories: [源码分析]
description: 介绍了locust的架构及其核心类
keywords: locust, 架构, 核心类
---

## 基本介绍
Locust是开源、使用Python开发、基于事件、支持分布式并且提供Web UI进行测试执行和结果展示的性能测试工具。

Locust的主要特性有两个：  
* 模拟用户操作:支持多协议，Locust可以用于压测任意协议类型的系统  
* 并发机制:摒弃了进程和线程，采用协程（gevent）的机制，单台测试机可以产生数千并发压力

Locust使用了以下几个核心库：

```
1) gevent
	gevent是一种基于协程的Python网络库，它用到Greenlet提供的，封装了libevent事件循环的高层同步API。
2) flask
	Python编写的轻量级Web应用框架。
3) requests
	Python Http库
4) msgpack-python
	MessagePack是一种快速、紧凑的二进制序列化格式，适用于类似JSON的数据格式。msgpack-python主要提供MessagePack数据序列化及反序列化的方法。
5) pyzmq
	pyzmq是zeromq(一种通信队列)的Python实现,主要用来实现Locust的分布式模式运行
```

## 系统架构及对比
我们知道，完整的压测系统应该包含以下组件：
![压测系统](http://processon.com/chart_image/5ddbf50ee4b08b68c8e1e203.png)

相比于主流的压测系统LoadRunner和Jmeter，Locust是一个更为纯粹的开源压测系统，框架本身的结构并不复杂，甚至只提供了最基础的组件，但也正因为如此，Locust具有极高的可编程性和扩展性。对于测试开发同学来说，可以比较轻松地使用Locust实现对任意协议的压测。

### 工具对比

|  | LoadRunner | Jmeter | Locust |
| ------ | ------ | ----- | ----- |
| 压力生成器 | √ | √ | √ |
| 负载控制器 | √ | √ | √ |
| 系统资源控制器 | √ | x | x |
| 结果采集器 | √ | √ | √ |
| 结果分析器 | √ | √ | √ |

上表简单展示了几个工具包含的压测组件，Locust的架构非常简单，部分组件的实现甚至都不完善，比如结果分析器，Locust本身并没有很详细的测试报告。但这并不妨碍它成为优秀的开源框架。

### Locust的架构

* locust架构上使用master-slave模型，支持单机和分布式
* master和slave使用 ZeroMQ 协议通讯
* 提供web页面管理master，从而控制slave，同时展示压测过程和汇总结果
* 可选no-web模式（一般用于调试）
* 基于Python本身已经支持跨平台

![](http://processon.com/chart_image/5ddbf7d3e4b034050df19b82.png)

为了减少CPython的GIL限制，充分利用多核CPU，建议单机启动CPU核数的slave（多进程）。

### 核心类
上面展示的Locust架构，从代码层面来看究竟是如何实现的呢？下面我们就来窥视一番：
![](http://processon.com/chart_image/5dd80265e4b0da22410e0d7f.png)

简单来说，Locust的代码分为以下模块：

* Runner-执行器，Locust的核心类，定义了整个框架的执行逻辑，实现了Master、Slave等执行器
* Locust-压测用例，提供了HttpLocust压测http协议，用户可以定义事务，断言等，也可以实现特定协议的Locust
* Socket-通信器，提供了分布式模式下master和slave的交互方式
* RequestStats-采集、分析器，定义了结果分析和数据上报格式
* EventHook-事件钩子，通过预先定义的事件使得我们可以在这些事件发生时(比如slave上报)做一些额外的操作

下面我们看看核心类的主要成员变量和方法

![](http://processon.com/chart_image/5dda4bd8e4b08b8173c61124.png)

* 用户定义的Locust类作为LocustRunner的locust_classes传入
* master的client_listener监听client消息
* slave的worker方法监听master消息
* slave的stats_reporter方法上报压测数据，默认3s上报一次
* slave的start_hatching启动协程，使用分配的并发数开始压测
* TaskSet和Locust持有client，可以直接发起客户端请求，client可以自己实现，Locust只实现了HttpLocust

看完上面的类图，是不是觉得Locust非常的简单呢？实际上，如果不是对Locust做二次开发，只是使用，那么基本上只会使用到Locust-压测用例相关的类，还有就是EventHook-事件钩子。当然，如果掌握了整个Locust的架构和核心类，那么对它进行二次开发会变得更简单，同时，日常的使用也会更加得心应手，Have fun！









