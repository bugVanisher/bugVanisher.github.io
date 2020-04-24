---
layout: post
title: 使用boomer压测grpc协议
categories: [性能测试]
description: grpc的压测实践以及boomer的压测应用实例
keywords: grpc, boomer, 性能测试
published: true
---

>由于工作原因，需要对grpc协议进行压测，在网上找了一圈，并没有找到可以直接引用的资料，为了得到较好的并发能力，我结合Locust+boomer实现了grpc协议的压测。

## grpc服务

[gRPC](http://www.grpc.io/)是一个高性能、通用的开源RPC框架，其由Google主要面向移动应用开发并基于HTTP/2协议标准而设计，基于ProtoBuf(Protocol Buffers)序列化协议开发，且支持众多开发语言。  

gRPC具有以下重要特征：

* 强大的IDL特性 RPC使用ProtoBuf来定义服务，ProtoBuf是由Google开发的一种数据序列化协议，性能出众，得到了广泛的应用。
* 支持多种语言 支持C++、Java、Go、Python、Ruby、C#、Node.js、Android Java、Objective-C、PHP等编程语言。 
* 基于HTTP/2标准设计

官网上有非常多语言的快速入门，为了演示跨语言调用，且boomer是基于go语言的，所以我演示的案例是go->python。首先根据官方的指引，起一个helloworld的grpc服务。

[https://grpc.io/docs/quickstart/python/](https://grpc.io/docs/quickstart/python/) 根据快速入门，起一个Python的grpc服务。

```sh
-> % python greeter_server.py                                                                                                                                          

```

为了验证服务是否正常启动了，我们直接使用greeter_client.py验证一下：

```sh
-> % python greeter_client.py
Greeter client received: Hello, you!
```

## 序列化和反序列化
为了从boomer侧发起请求，首先需要对请求和响应做序列化与反序列化，在阅读grpc的源码后，整理如下：

```go
// ProtoCodec ...
type ProtoCodec struct{}

// Marshal ...
func (s *ProtoCodec) Marshal(v interface{}) ([]byte, error) {
	return proto.Marshal(v.(proto.Message))
}

// Unmarshal ...
func (s *ProtoCodec) Unmarshal(data []byte, v interface{}) error {
	return proto.Unmarshal(data, v.(proto.Message))
}

// Name ...
func (s *ProtoCodec) Name() string {
	return "ProtoCodec"
}
```

## 服务调用
我的想法是提供一套调用grpc服务的通用client，所以调用服务+方法时需要是动态的，正好grpc提供了Invoke方法可以满足这一点，接下来定义一个Requester结构体。

```go
// Requester ...
type Requester struct {
	addr      string
	service   string
	method    string
	timeoutMs uint
	pool      pool.Pool
}
```

Requester中定义两个方法，一个是获取真实的调用方法getRealMethodName，一个是具体的调用Call，其中Call是暴露给外层调用的。

![]({{ site.url }}/assets/grpc/grequester.jpg)



```go
// getRealMethodName
func (r *Requester) getRealMethodName() string {
	return fmt.Sprintf("/%s/%s", r.service, r.method)
}
```

Call方法核心代码

```go
if err = cc.(*grpc.ClientConn).Invoke(ctx, r.getRealMethodName(), req, resp, grpc.ForceCodec(&ProtoCodec{})); err != nil {
		fmt.Fprintf(os.Stderr, err.Error())
		return err
	}
```

## 连接池
如http1.1的Keep-Alive，在高并发下需要保持grpc连接以提高性能，所以需要实现一个grpc的连接池管理，这也是Requester结构体中pool职责。

Requester实例化时初始化连接池：

```go
// NewRequester ...
func NewRequester(addr string, service string, method string, timeoutMs uint, poolsize int) *Requester {
	//factory 创建连接的方法
	factory := func() (interface{}, error) { return grpc.Dial(addr, grpc.WithInsecure()) }

	//close 关闭连接的方法
	closef := func(v interface{}) error { return v.(*grpc.ClientConn).Close() }

	//创建一个连接池： 初始化5,最大连接200,最大空闲10
	poolConfig := &pool.Config{
		InitialCap: 5,
		MaxCap:     poolsize,
		MaxIdle:    10,
		Factory:    factory,
		Close:      closef,
		//连接最大空闲时间，超过该时间的连接 将会关闭，可避免空闲时连接EOF，自动失效的问题
		IdleTimeout: 15 * time.Second,
	}
	apool, _ := pool.NewChannelPool(poolConfig)
	return &Requester{addr: addr, service: service, method: method, timeoutMs: timeoutMs, pool: apool}
}
```

这里使用了开源库 [pool](https://github.com/silenceper/pool) 来做grpc的连接管理。在Call方法中每次发起请求前在连接池中获取一个连接，调用完成后释放回连接池中。

## 脚本编写
接下来就是编写boomer脚本了，我们需要两个文件，一个是pb结构的请求和响应，一个是main.go

### a、基于.proto生成供go使用的.go文件
grpc使用PB结构传输消息，.proto文件定义了PB数据，使用protoc工具可以生成直接给不同语言使用的数据结构和接口定义文件，如下

```sh
-> % protoc helloworld.proto --go_out=./
```

### b、编写压测脚本并引入.go文件对象
在helloworld例子中存在两个PB对象，分别是HelloRequest、HelloReply，python暴露的rpc服务和接口分别为helloworld.Greeter和SayHello，所以调用方式如下：

```go
startTime := time.Now()

// 构建请求对象
request := &HelloRequest{}
request.Name = req.Name

// 初始化响应对象
resp := new(HelloReply)
err := client.Call(request, resp)

elapsed := time.Since(startTime)
```
完整的文件地址请看 [main.go](https://github.com/bugVanisher/boomer/blob/master/examples/rpc/main.go)

### c、调试脚本
使用boomer的 --run-tasks 调试脚本

```sh
-> % cd examples/rpc
-> % go run *.go -a localhost:50051 -r '{"name":"bugVanisher"}' --run-tasks rpcReq
2020/04/24 21:31:11 {"name":"bugVanisher"}
2020/04/24 21:31:11 Running rpcReq
2020/04/24 21:31:11 Resp Length: 29
2020/04/24 21:31:11 message:"Hello, bugVanisher!"
```

至此，基于boomer的grpc压测脚本已经完成了，剩下的就是结合Locust对被测系统进行压测了，我这里就不赘述了。

## 总结
本文展示了如何压测一个官方Demo的grpc服务接口，这仅是一个演示，真实的业务一般会针对grpc框架做封装，也许不同的语言有各自完整的一套开源框架了。需要注意的地方是，不同的框架下，我们Invoke时真实的method可能有所不同，要根据实际情况做修改。

结合连接池和boomer，压测脚本能产生非常好的施压能力，这也体现了go语言的强大。如果想实现非http协议的压测同时又想拥有不错的并发施压能力，boomer是一个不错的选择。

