---
layout: post
title: 使用whistle解析Protobuf协议数据
categories: [网络]
description: 抓包解析proto协议数据
keywords: 代理，解析，protobuf
published: true
---



我所在的业务对protobuf的应用比较普遍，无论在websocket长连接的数据帧亦或是通用http请求过程中都喜欢使用protobuf进行序列化，这算是一种私有协议吧。虽然在私密性和性能上有一定的提升，但是测试的时候就非常难受了，所以我的目标很简单，就是能够自动将这些pb序列化的数据反序列化成人可以阅读的文本，比如JSON格式。

## 工具选择

显然，对于私有协议的抓包解包需要抓包工具支持插件开发，我平时习惯用burpsuite，虽然它提供http的插件接口，但是它并没有提供websocket插件开发的接口（试过翻看文档，着实没找到。。），于是我将目光投向了国人开发的工具——whistle，刚好它提供了一些插件开发的文档，虽然最近更新也是几年前了，但是总归在设计上是支持的。插件开发的流程和原理我就不赘述了，可以看[这里](https://wproxy.org/whistle/plugins.html)。

whistle官方提供了插件开发的 [实例代码](https://github.com/whistle-plugins) ，令人头疼的是它的官方插件文档写得比较早，插件实例推出后没有更新相应的文档，而且我的场景实例有些差异，于是，开始了艰难的尝试之路。。。

## 解开websocket数据帧

<img src="https://bugvanisher.cn/images/static/image-20210202142222003.png" alt="websocket上下行数据" style="zoom:80%;" />

第一个尝试是解websocket的数据帧，我将目光投向了 [whistle.script](https://github.com/whistle-plugins/whistle.script), 因为需要反序列化protobuf，所以要加载*.proto文件，找到了js里操作pb的库 [protobufjs](https://www.npmjs.com/package/protobufjs), 查看官方文档后，可以使用如下代码反序列化ws数据帧。

```javascript

ProtoBuf.load(msgProto, function (err, root) {
    if (err)
        throw err;

    // Obtain a message type
    let AwesomeMessage = root.lookupType("msg.Msg");

    // Decode an Uint8Array (browser) or Buffer (node) to a message
    let message = AwesomeMessage.decode(data);

    // ... do something with message
    console.log(`Client: ${JSON.stringify(message)}`);
});

```

说明：msgProto是本地的*.proto文件地址，msg.Msg分别是proto文件定义的包名和Message， .decode方法将数据反序列化即从二进制的pb转为js对象。

这里有个小插曲，websocket包里有两层pb，如下：

```protobuf
// 消息系统消息
message Msg {
    optional uint64 id = 1; // 消息ID
    optional uint32 kind = 2; // 推送类型，详见: Kind
    optional uint32 type = 3; // 消息类型，详见：Type
    optional uint64 to_uid = 4; // UID
    optional uint64 session_id = 5; // session ID，必填
    optional bytes content = 6; // 消息内容（消息结构体pb二进制流）
    optional uint64 timestamp = 7; // 消息发送时间
}
```

content字段就是内层pb序列化后的字节流，反序列化的时候根据外层的type选择proto对应的Message类型进行反序列化，所以需要加一层判断。代码类似如下：

```javascript
function getContent(msgType, root, serialContent) {
    let content = null;
    switch (msgType) {
        case 1001:
            content = root.lookupType("msg.LikeCntMsg");
            break;
        case 1002:
            content = root.lookupType("msg.CcuMsg");
            break;
        ...
    }
    if (content != null) {
        return content.decode(serialContent);
    } else {
        return serialContent;
    }
```

这里根据msgType将序列化的content字段反序列化为js对象然后返回，剩下的内容就是构造最外层的对象，再打印在console中，完整代码如下：

```javascript
exports.handleWebSocket = async (ws, connect) => {
    // 将请求继续转发到目标后台，如果不加则直接响应
    const res = await connect();
    // 获取客户端的请求数据
    ws.on('message', (data) => {
            ProtoBuf.load(msgProto, function (err, root) {
                if (err)
                    throw err;

                // Obtain a message type
                let AwesomeMessage = root.lookupType("msg.Msg");

                // Decode an Uint8Array (browser) or Buffer (node) to a message
                let message = AwesomeMessage.decode(data);

                // ... 打印客户端发送出去的数据
                console.log(`Client: ${JSON.stringify(message)}`);
            });
            // 可以修改后再发送到Server
            res.send(data);
        }
    );
    res.on('message', (data) => {
        ProtoBuf.load(msgProto, function (err, root) {
            if (err)
                throw err;
            // Obtain a message type
            let AwesomeMessage = root.lookupType("msg.Msg");
            // Decode an Uint8Array (browser) or Buffer (node) to a message
            let message = AwesomeMessage.decode(data);
            let con = getContent(message.type, root, message.content);
            // 重新构造最外层的对象结构，直接将转换的con覆盖message.content会有问题
            let frame = {
                id: parseInt(message.id).toString(),
                kind: message.kind,
                type: message.type,
                toUid: parseInt(message.toUid).toString(),
                sessionId: parseInt(message.sessionId).toString(),
                content: con,
                timestamp: parseInt(message.timestamp).toString()
            }
            // 在script的Console打印出服务端发送过来的数据
            console.log(`Server: ${JSON.stringify(frame)}`);

        });
        // 可以修改后再发送到Server
        ws.send(data);
    });
// 接收通过whistle.script页面Console的dataSource.emit('toSocketClient', {name: 'toSocketClient'})的数据
    ws.dataSource.on('toSocketClient', (data) => {
        ws.send(data);
    });
// 接收通过whistle.script页面Console的dataSource.emit('toSocketServer', {name: 'toSocketClient'})的数据
    ws.dataSource.on('toSocketServer', (data) => {
        res.send(data);
    });

};
```

打印出的内容如下：

```shell
Server: {"id":"371","kind":185,"type":1001,"toUid":"0","sessionId":"0","content":{},"timestamp":"0"}
Server: {"id":"1253034067729926","kind":0,"type":1002,"toUid":"32","sessionId":"6799373","content":{"ccu":"1"},"timestamp":"1612239130113"}
Client: {"kind":372,"type":2001}
```



## 解开http请求Body

使用 whistle.script 只能在插件模块的console打印出明文数据，对于websocket这种长连接看连续的不同消息还算方便，但对于单个的http请求来说就很麻烦了，能不能直接在 Inspectors 里查看呢？于是我发现了 [whistle.pipe](https://github.com/whistle-plugins/examples/tree/master/whistle.test-pipe) 这个插件，它可以用来解析请求和响应数据并在Inspector模块中查看。于是我按照whistle的插件开发方式，使用 lack 创建了 reqRead、reqWrite 而不是 resRead、resWrite（与官方实例的解析响应内容不同，我需要解析的是请求的body）。

我要解析的是一个上报数据接口，它的请求body是pb序列化的，message 定义如下：

```protobuf
// EventList即是request body
message EventList {repeated Event events = 1;}

message Event {
  optional Header header = 1;
  optional bytes body = 2;
}

message Header {
  optional uint32 id = 1; // 事件ID，事件类型唯一，详见：EventID
  optional uint32 scene_id = 2; // 场景ID，业务场景唯一，详见：SceneID
  optional uint64 uid = 3;      // 用户ID
  optional string device_id = 4;    // 设备ID
  optional string device_model = 5; // 设备类型，例如 “iPhone6 Plus”
  optional uint32 os = 6;           // 操作系统  0：Android 1: iOS 2：PCWeb
  optional string os_version = 7;     // 系统版本号，例如 “10.3.1”
  optional string client_version = 8; // 客户端版本号，例如 “2.39”
  optional string client_ip = 9;      // 客户端IP
  optional uint32 network = 10; // 网络类型  1: WI-FI；2：蜂窝; 3: 2G；4: 3G；5: 4G;
  optional string country = 11;     // 地区
  optional string ua = 12;          // User Agent
  optional string sdk_version = 13; // 上报SDK版本
  ...
}
```

服务端会根据header的id等字段来反序列化body，场景跟websocket的两层pb类似。整个解析的思路就是在reqRead的时候将客户端已序列化的数据反序列出来，并输出json，在reqWrite真正发出请求时将数据序列回去再发给真正的服务器，这样明文的信息就可以直接在 Inspector 中查看。我们先看看解析前后的情况：

**解析前**

<img src="https://bugvanisher.cn/images/static/image-20210202145951020.png" alt="解析前" style="zoom:50%;" />



**解析后**

<img src="https://raw.githubusercontent.com/bugVanisher/images/master/static/image-20210202185835789.png" alt="image-20210202185835789" style="zoom:50%;" />



下面就来看看reqRead 和 reqWrite应该如何编写：

```javascript
// reqRead.js
let {getBodyObj, pb, event} = require("./base");

let ProtoBuf = require("protobufjs");

module.exports = (server/* , options */) => {
  server.on('request', (req, res) => {
    let body;
    req.on('data', (data) => {
      body = body ? Buffer.concat([body, data]) : data;
    });
    req.on('end', () => {
      if (body) {
        ProtoBuf.load([pb, event], function (err, root) {
          if (err)
            throw err;

          // Obtain a message type
          let AwesomeMessage = root.lookupType("event.EventList");

          // Decode an Uint8Array (browser) or Buffer (node) to a message
          let messageList = AwesomeMessage.decode(body);

          let events = []
          for (const singleEvent of messageList.events) {
            let body = getBodyObj(singleEvent.header.id, root, singleEvent.body)
            events.push({header: singleEvent.header, body: body});
          }
          res.end(JSON.stringify(events));
        });

      } else {
        res.end();
      }
    });
  });
};
```

```javascript
// reqWrite.js
let {getBodyBytes, pb, event} = require("./base")
let ProtoBuf = require("protobufjs");

module.exports = (server/* , options */) => {
  server.on('request', (req, res) => {
    let body;
    req.on('data', (data) => {
      body = body ? Buffer.concat([body, data]) : data;
    });
    req.on('end', () => {
      if (body) {
        ProtoBuf.load([pb, event], function (err, root) {
          if (err)
            throw err;
          let events = JSON.parse(body);
          let eList = [];
          for (const singleEvent of events) {
            let bodyBytes = getBodyBytes(singleEvent.header.id, root, singleEvent.body);
            let EventMsg = root.lookupType("event.Event");
            let message = EventMsg.create({
              header: singleEvent.header,
              body: bodyBytes
            });
            eList.push(message);
          }
          let EventList = root.lookupType("event.EventList");
          let eventList = EventList.create({
            events: eList
          });
          let buffers = EventList.encode(eventList).finish();
          res.end(buffers);
        });
      } else {
        res.end();
      }
    });
  });
};	
```

```javascript
// base.js
const getBodyObj = (eventId, liveRoot, bodyBuffer) => {
  let Body;
  Body = getPB(eventId, liveRoot)
  if (Body != null) {
    return Body.decode(bodyBuffer);
  } else {
    return bodyBuffer;
  }
}

const getBodyBytes = (eventId, liveRoot, bodyStr) => {
  let Body;
  Body = getPB(eventId, liveRoot)
  if (Body != null) {
    // get proto buffer
    return Body.encode(bodyStr).finish()
  } else {
    return bodyStr;
  }
}

function getPB(eventId, liveRoot) {
  let Body = null;
  switch (eventId) {
    case 10001:
      Body = liveRoot.lookupType("live_event.StreamExceptionEvent");
      break;
    case 10002:
      Body = liveRoot.lookupType("live_event.StreamStartEvent");
      break;
    ...
  }
  return Body;
}

exports.pb = "~/event.proto"
exports.event = "~/live_event.proto"

exports.getBodyObj = getBodyObj;
exports.getBodyBytes = getBodyBytes;

```



## 写在最后

由于时间和精力有限，并没有深入地研究whistle的整个插件开发机制，靠着官方实例插件代码，目前只是做到了WebSocket、HTTP包解析，不能够手动篡改，比如在Composer中构造请求，但已经满足了最迫切的需求，如果以后有时间了再研究分享。

Whistle本身是一个非常棒的工具，无论是通用的websocket、http还是私有协议，都可以抓包解包，甚至是改包，但是要吐槽下whistle的插件开发文档，真的有些过时了，而且也写的不够清楚，开发过程遇到的问题很多时候不知道怎么求解，尤其对不太熟悉nodejs的人！

