---
layout: post
title: Runner的状态与通信机制——locust源码分析
categories: [源码分析]
description: 介绍了locust的runner的状态与通信机制
keywords: locust, runner
---

上一篇文章讲了[Locust的架构和核心类]({{ site.baseurl }}/2019/03/22/the-structure-of-locust/)，那么接下来应该学习什么呢？我认为应该是：Runner的状态和通信机制。
为什么呢？我们知道Locust等压测工具支持分布式压测，就是说理论上可以通过不断添加压力机(slave)提高并发数量，这个机制让使用者可以自由地增减机器资源。

### Runner状态机

在分布式场景下，除了数据一致性，状态同步也是非常重要的。在Locust的master-slave架构下，需要管理master和slave的状态，不仅为了控制压测的开始或停止，也是为了掌握当前的压力机情况。那么都有哪些状态？

| 状态 | 说明 | 
| ------ | ------ |
| ready | 准备就绪，master和slave启动后默认状态 |
| hatching | 正在孵化压力机，对master来说正在告诉slave们开始干活，对slave来说是过渡状态，因为它们马上要running |
| running | 正在压测 |
| cleanup | 当发生GreenletExit时的状态，一般不会出现 |
| stopping | 表示正在通知slave们停止，只有master有这个状态 |
| stopped | 压测已经停止 |
| missing | 状态丢失，只有slave有的状态，默认3秒如果master没有收到slave的心跳就会认为它missing了，一般是进程没有正常退出导致 |

Runner的状态不多，但是在压测过程中起到非常重要的作用，状态之间是按约定的方式进行扭转的，我们使用Locust的web界面管理master的状态，master根据我们的操作通过通信机制推进slave的状态。
![](http://processon.com/chart_image/5dd7f966e4b052b7c58c33d4.png)

### 通信机制
Master与Slave之间是通过Zeromq建立的TCP连接进行通信的（一对多）。

ZeroMQ（简称ZMQ）是一个基于消息队列的多线程网络库，其对套接字类型、连接处理、帧、甚至路由的底层细节进行抽象，提供跨越多种传输协议的套接字。

ZMQ是网络通信中新的一层，介于应用层和传输层之间（按照TCP/IP划分），其是一个可伸缩层，可并行运行，分散在分布式系统间。

master与各个slave各维持一个TCP连接，在每个连接中，master下发的命令，slave上报的信息等自由地的传输着。

#### 消息格式

```python
class Message(object):
    def __init__(self, message_type, data, node_id):
        self.type = message_type
        self.data = data
        self.node_id = node_id

    def serialize(self):
        return msgpack.dumps((self.type, self.data, self.node_id))

    @classmethod
    def unserialize(cls, data):
        msg = cls(*msgpack.loads(data, raw=False))
        return msg
```

其中message_type指明消息类型，data是实际的消息内容，node_id指明机器ID。Locust使用[msgpack](https://msgpack.org/)做序列化与反序列化处理。

#### 消息类型

| 序号 | message_type | 发送者 | data格式 | 发送时机 |
| ------ | ------ | ------ | ------ | ------ |
| 1 | client_ready | slave | 空 | slave启动后或压测停止完成 |
| 2 | hatching | slave | 空 | 接受到master的hatch先发送一个确认 |
| 3 | hatch_complete | slave | {"count":n} | 所有并发用户已经孵化完成 |
| 4 | client_stopped | slave | 空 | 停止所以并发用户后 |
| 5 | heatbeat | slave | {'state': x} | 默认每3秒上报一次心跳 |
| 6 | stats | slave | {'stats':[],'stats_total':{},'errors':{},user_count:x} | 每3秒上报一次压测信息 |
| 7 | exception | slave | {"msg" : x, "traceback" : x} | 出现异常 |
| 8 | hatch | master | {"hatch_rate":x,         "num_clients":x,"host":x} | 开始swarm |
| 9 | stop | master | 空 | 点击stop |
| 1 | quit | master，slave | 空 | 手动或其他方式退出的时候

可以看到上面有一种非常重要的消息类型——stats，压测的结果采集都封装在这个消息里，我将在下一篇文章分析Locust究竟是如何做结果采集和分析的。
