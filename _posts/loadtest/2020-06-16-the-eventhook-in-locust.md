---
layout: post
title: 事件钩子——locust源码分析
categories: [源码分析]
description: 介绍locust的事件钩子-eventHook
keywords: locust, 事件钩子, 监听器
published: false
---

> 本文讲解的eventHook基于Locust的1.x版本。

## 基本介绍

Locust的event.py模块包含了两个类，一个是事件钩子定义类EventHook，一个是事件钩子类型类Events，为不同的事件提供hook。事件处理函数注册相应的hook以后，我们可以很方便的的基于event触发处理函数，实现事件驱动。

EventHook定义了三个方法：

```python
def add_listener(self, handler):
    self._handlers.append(handler)
    return handler

def remove_listener(self, handler):
    self._handlers.remove(handler)
    
def fire(self, *, reverse=False, **kwargs):
    if reverse:
        handlers = reversed(self._handlers)
    else:
        handlers = self._handlers
    for handler in handlers:
        handler(**kwargs)
```

可以看到add_listener、remove_listener的作用是注册或删除监听函数，fire方法是触发处理函数。EventHook的定义相比Locust 0.x版本有较大改变。

Events中包含了7个事件钩子，分别是：

| 序号 | 事件钩子                 | 入参                                                         | 触发时机                                                     |
| ---- | ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | request_success          | :param request_type: Request type method used<br>:param name: Path to the URL that was called (or override name if it was used in the call to the client) <br>:param response_time: Response time in milliseconds <br>:param response_length: Content-length of the response | 当一个请求完全成功的时候                                     |
| 2    | request_failure          | :param request_type: Request type method used <br>:param name: Path to the URL that was called (or override name if it was used in the call to the client) :param response_time: Time in milliseconds until exception was thrown <br>:param response_length: Content-length of the response <br>:param exception: Exception instance that was thrown | 当一个请求失败的时候，默认statusCode非200为Failure，可自定义 |
| 3    | user_error               | :param user_instance: User class instance where the exception occurred <br>:param exception: Exception that was thrown <br>:param tb: Traceback object (from sys.exc_info()[2]) | 在User class中出现未捕获异常的时候                           |
| 4    | report_to_master         | :param client_id: The client id of the running locust process. <br>:param data: Data dict that can be modified in order to attach data that should be sent to the master. | worker模式会定时上报数据，每次上报时都会触发一次             |
| 5    | worker_report            | :param client_id: Client id of the reporting worker <br>:param data: Data dict with the data from the worker node | master模式每次收到一个worker上报的一条数据就会触发一次       |
| 6    | hatch_complete           | :param user_count: Number of users that was hatched          | 分配给单个worker的所有并发用户孵化完成是触发一次             |
| 7    | quitting                 | :param environment: Environment instance                     | 当Locust进程即将退出时触发一次                               |
| 8    | init                     | :param environment: Environment instance                     | Locust启动时触发，为了让使用者方便地访问Envirionment对象，在1.x版本后此方法非常重要 |
| 9    | init_command_line_parser | :param parser: ArgumentParser instance                       | 为了方便用户添加Locust命令自定义参数                         |
| 10   | test_start               | 无                                                           | 开始压测时触发，只在master侧触发                             |
| 11   | test_stop                | 无                                                           | 停止压测时触发，只在master侧触发                             |

## 事件钩子的原理



## 如何使用钩子

