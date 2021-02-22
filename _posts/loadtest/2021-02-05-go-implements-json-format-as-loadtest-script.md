---
layout: post
title: Go实现json格式定义http协议压测脚本
categories: [性能测试]
description: go实现json格式的压测脚本编写
keywords: 测试平台,Go,Json,Http,脚本
published: true
---

前段时间，我主导推动组里实现了一套基于Locust+boomer的通用的压测平台，主要目的是满足我们组内的各种压测场景，比如grpc、websocket、webrtc、http等协议的压测场景。正好我们公司的技术栈以go为主，我们可以轻松地使用go编写脚本，通过公司的部署平台编译打包后横向扩缩施压集群，可以说解决了各种压测需求。但是我们发现，尽管自己编写脚本非常自由，但是对不了解平台、不了解Go的同学来说，使用成本是比较大的，尤其是首次接触，因此我开始思考如何简化脚本的编写和部署。

## 从http开始

来源于httprunner和公司其他Group的工具的灵感，我想到用json的方式去定义http压测场景，然后用go去解析执行，可以预见的是这种方式的压测性能不如直接写代码，但是如果可以通过可接受的性能损耗来换取更简单的接入方式，更统一的使用方式，也是极好的，毕竟我们不缺机器。针对http协议，以下大概梳理了一下需要实现的能力。

![http压测模板](http://assets.processon.com/chart_image/600687fce0b34d45d1616f92.png)

简单说明下：

- 多接口

  很好理解，压测的时候需要满足多个接口按一定的比例同时压测。在某些特殊场景下，可能还存在接口依赖的问题，这也要考虑到。

- 登录态

  压测的接口可能有登录校验，那么压测的时候需要带上登录态，如果能打通账号平台自动批量生成登录态就方便很多了。

- 参数化

  定义的脚本需要提供参数化能力，总不能所有参数写死，比如动态生成时间戳，ID，变长字符串等等，如果简单的参数生成无法满足，用户自己上传也是挺好的。

- 校验

  响应内容校验是接口测试很重要的部分，在压测场景也是一样的。

## 定义Json结构

接下来，定义Json结构，尽量去满足上面所描述的需求。http协议，无非就是三个部分，body、header、url，因此每一个接口需要包含这个三个字段，当然，名字是必不可少的，还有一个非常重要的字段，就是校验字段validator，下面就来看看这个Json应该是什么样子。
{% raw %}

```javascript
{
    "debug": true,
    "domain": "https://postman-echo.com",
    "header": {},  
    "declare": [
        "{{ $sessionId := getSid }}"
    ],
    "init_variables": {
        "roomId": 1001,
        "sessionId": "{{ $sessionId }}",
        "ids": "{{ $sessionId }}"
    },
    "running_variables": {
        "tid": "{{ getRandomId 5000 }}"
    },
    "func_set": [
        {
            "key": "getTest",
            "method": "GET",
            "url": "/get?name=gannicus&roomId={{ .roomId }}&age=10&tid={{ .tid }}",
            "body": "{\"timeout\":10000}",
            "validator": "{{ and  (eq .http_status_code 200) (eq .args.age (10 | toString )) }}"
        },
        {
            "key": "postTest",
            "method": "POST",
            "header": {
                "Cookie": "{{ .tid }}",
                "Content-Type": "application/json"
            },
            "url": "/post?name=gannicus",
            "body": "{\"timeout\":{{ .tid }}, \"retry\":true}",
            "validator": "{{ and  (eq .http_status_code 200) (eq .data.timeout (.tid | toFloat64 ) ) (eq .data.retry false) }}"
        }
    ]
}
```
{% endraw %}
func_set应该挺好理解的，这里解释一下declare、init_variables、running_variables：

- declare
  {% raw %}
  这个字段是为了声明变量的，比如在init或running变量中都可以引用这个变量，声明方式如：

  ```json
  [
  	"{{ $sessionId := getSid }}",
  	"{{ $id := 100100 }}"
  ]
  ```


  {% endraw %}

- init_variables

  初始化变量，只初始化一次，可以是常量，也可以从模板函数中获取，如：
  {% raw %}

  ```json
  {
          "roomId": 1001,
          "sessionId": "{{ $sessionId }}",
          "ids": "{{ now }}"
  }
  ```
  {% endraw %}

- running_variables

  运行时变量，每一个请求发起前都会去构造参数，因此不建议常量定义在这里。
  
  {% raw %}
  
  ```json
  {
      	"tid": "{{ getRandomId 5000 }}"
  }
  ```
  
  {% endraw %}

## 解析流程

想要利用boomer，那就需要想办法生成boomer.Task，它的结构如下：

```go
type Task struct {
	// The weight is used to distribute goroutines over multiple tasks.
	Weight int
	// Fn is called by the goroutines allocated to this task, in a loop.
	Fn   func()
	Name string
}
```

核心就是得到这个执行函数Fn，思路就是分别根据init和running变量定义，为func_set中声明的每个请求分别定义一个匿名函数，函数中去动态生成变量，然后发起真实请求，最后根据每个请求声明的validator进行断言。整个执行流程如下：

![执行流程](http://assets.processon.com/chart_image/601a9264e401fd1b8db5db28.png)

## 实现原理

go的原生库中就有模板相关的库text/template,我直接使用模板库实现了这套解析逻辑，包括参数的生成，模板方法、断言，整个json脚本的语法都是基于go的模板库的。感兴趣的朋友可以查看：

- [官方模板库](https://golang.org/pkg/text/template/)
- [Go 语言标准库text/template 包深入浅出](https://juejin.cn/post/6844903762901860360)

## 如何断言

因为断言这部分非常重要，所以单独讲。上面已经说过断言是通过模板来实现的，所以要使用断言就要掌握基本的模板语法。

模板库内置了比较和逻辑方法所以可以直接使用，比如比较http状态码:

{% raw %}

```
"validator": "{{ eq .http_status_code 200 }}"
```

{% endraw %}

再比如多个比较:
{% raw %}

```
"validator": "{{ and  (eq .http_status_code 200) (eq .data.timeout (.tid | toFloat64 ) ) (eq .data.retry false) }}"
```

{% endraw %}

可能你也已经注意到了有一个toFloat64, 它是一个模板自定义函数，这里是为了做类型转换。

此外，你也可以看到基于go模板库，访问变量变得非常简单，比如上面的.data.timeout,它对应响应中的内容类似如下：

```json
{
	"data":{
		"timeout": 1000
	}
}
```

这样我们就可以比较响应json中的任何字段了。

## 写在最后

简单做了一下压力测试，对比不使用模板解析和使用模板解析的情况，模板解析在CPU密集型的场景下性能大概是直接写脚本编译的三分之一，如果不是CPU密集型应该可以去到二分之一，因此我暂时不优化了。令人惊喜的是这个模板解析也可以扩展成为接口测试的编写方式，类似httprunner。

目前只是做了一个Demo，还没正真集成到我们的压测平台，还是挺令人期待呀。

源码地址：[https://github.com/bugVanisher/go-httpwrapper](https://github.com/bugVanisher/go-httpwrapper)

