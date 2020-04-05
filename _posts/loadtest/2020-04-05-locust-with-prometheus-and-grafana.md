---
layout: post
title: 实战|Locust+Prometheus+Grafana 搭建性能监控平台
categories: [测试开发]
description: Locust性能监控平台
keywords: Locust, 监控平台, Prometheus, Grafana
published: true
---

## 需求背景
当我们使用Locust做性能压测的时候，压测的过程和展示如下：
![]({{ site.url }}/assets/locust/statistics.jpg) 
![]({{ site.url }}/assets/locust/charts.jpg) 

其中波动图是非持久化存储的，也就是刷新后，波动图就清空了。尽管Statistics中显示的信息比较完整，但是都是瞬时值，并不能体现出时序上的变化。像Failures、Exceptions、Slaves分在不同的tag查看起来也比较麻烦。Locust的测试报告也只有简单的csv文件，需要下载。			

从上面我们可以看到Locust虽然提供了跨平台的web模式的性能监控和展示，但是有以下明显缺陷：

* rps、平均响应时间波动图没有持久化存储，刷新后遍丢失
* 整体统计信息只是表格的形式，不能体现波动时序
* 测试报告过于简陋且只有文字版，只能下载存档

## 需求方案
方案其实很多选择，为了减少投入成本和最大化利用现用的开源工具，可以选择：

* Locust + InfluxDB + Grafana
* Locust + Prometheus + Grafana

我使用的是第二个方案，简单总结起来就是：

```
实现一个Locust的prometheus的exporter，将数据导入prometheus，然后使用grafana进行数据展示。
```

*不难发现Jmeter在网上有许多类似方案的介绍，但很遗憾的是我没有找到很好实现Locust监控平台的实现，所以只能自己实现了。*


## Docker环境
Docker环境不是必须的，但是用过都说好。我们这次实战是在docker中完成的，因为它实在是太方便了，如果你也想快速尝试一下本文的监控平台方案，建议先准备好docker环境。


## 编写locsut prometheus exporter
如Locust的官方文档所介绍的 [Extending Locust](https://docs.locust.io/en/stable/extending-locust.html) 我们可以扩展web端的接口，比如添加一个/export/prometheus 接口，这样Prometheus根据配置定时来拉取Metric信息就可以为Grafana所用了。这里需要使用Prometheus官方提供的client库，[prometheus_client](https://github.com/prometheus/client_python)，来生成符合Prometheus规范的metrics信息，具体的实现代码如下：

```python
#!/usr/bin/env python
# coding: utf-8

"""
    Created by bugVanisher on 2020-03-21
"""

from itertools import chain

import six
from flask import request, Response
from prometheus_client import Metric, REGISTRY, exposition

from locust import web, runners, stats as locust_stats

class LocustCollector(object):
    registry = REGISTRY

    def collect(self):
        # 只有在运行的时候才产生metric，这样在停止压测的时候prometheus就不会收集多余的数据.
        if runners.locust_runner and runners.locust_runner.state in ([runners.STATE_HATCHING, runners.STATE_RUNNING]):

            stats = []

            for s in chain(locust_stats.sort_stats(runners.locust_runner.request_stats),
                           [runners.locust_runner.stats.total]):
                stats.append({
                    "method": s.method,
                    "name": s.name,
                    "num_requests": s.num_requests,
                    "num_failures": s.num_failures,
                    "avg_response_time": s.avg_response_time,
                    "min_response_time": s.min_response_time or 0,
                    "max_response_time": s.max_response_time,
                    "current_rps": s.current_rps,
                    "median_response_time": s.median_response_time,
                    "avg_content_length": s.avg_content_length,
                    "current_fail_per_sec": s.current_fail_per_sec
                })

            # 这里的errors key是method+Name+Type哈希过的，相同method+Name的接口如果Type不一样会导致丢失部分数据（add_sample是被覆盖）
            errors = [e.to_dict() for e in six.itervalues(runners.locust_runner.errors)]

            metric = Metric('locust_user_count', 'Swarmed users', 'gauge')
            metric.add_sample('locust_user_count', value=runners.locust_runner.user_count, labels={})
            yield metric

            metric = Metric('locust_errors', 'Locust requests errors', 'gauge')
            for err in errors:
                metric.add_sample('locust_errors', value=err['occurrences'],
                                  labels={'path': err['name'], 'method': err['method']})
            yield metric

            is_distributed = isinstance(runners.locust_runner, runners.MasterLocustRunner)
            if is_distributed:
                metric = Metric('locust_slave_count', 'Locust number of slaves', 'gauge')
                metric.add_sample('locust_slave_count', value=len(runners.locust_runner.clients.values()), labels={})
                yield metric

            metric = Metric('locust_fail_ratio', 'Locust failure ratio', 'gauge')
            metric.add_sample('locust_fail_ratio', value=runners.locust_runner.stats.total.fail_ratio, labels={})
            yield metric

            metric = Metric('locust_state', 'State of the locust swarm', 'gauge')
            metric.add_sample('locust_state', value=1, labels={'state': runners.locust_runner.state})
            yield metric

            stats_metrics = ['avg_content_length', 'avg_response_time', 'current_rps', 'current_fail_per_sec', 'max_response_time',
                             'median_response_time', 'min_response_time', 'num_failures', 'num_requests']

            for mtr in stats_metrics:
                mtype = 'gauge'
                if mtr in ['num_requests', 'num_failures']:
                    mtype = 'counter'
                metric = Metric('locust_requests_' + mtr, 'Locust requests ' + mtr, mtype)
                for stat in stats:
                    # Aggregated stats method label is None, so name it as total
                    # locust change name Total to Aggregated since 0.12.1
                    if 'Aggregated' != stat['name']:
                        metric.add_sample('locust_stats_' + mtr, value=stat[mtr],
                                          labels={'path': stat['name'], 'method': stat['method']})
                    else:
                        metric.add_sample('locust_stats_' + mtr, value=stat[mtr],
                                          labels={'path': stat['name'], 'method': 'Aggregated'})
                yield metric


@web.app.route("/export/prometheus")
def prometheus_exporter():
    registry = REGISTRY
    encoder, content_type = exposition.choose_encoder(request.headers.get('Accept'))
    if 'name[]' in request.args:
        registry = REGISTRY.restricted_registry(request.args.get('name[]'))
    body = encoder(registry)
    return Response(body, content_type=content_type)

```
以上文件命名为exporter.py,下面编写一个Demo.py：

```python
#!/usr/bin/env python
# coding: utf-8


"""
    Created by bugVanisher on 2020-03-21
"""

from prometheus_client import REGISTRY

from exporter import LocustCollector
from locust import HttpLocust, TaskSet, task, between

# 注册收集器
REGISTRY.register(LocustCollector())

class NoSlowQTaskSet(TaskSet):

    def on_start(self):
        # 登录
        data = {"username": "admin", "password": "admin"}
        self.client.post("/user/login", json=data)

    @task(50)
    def getTables(self):
        r = self.client.get("/newsql/api/getTablesByAppId?appId=1")

    @task(50)
    def get_apps(self):
        r = self.client.get("/user/api/getApps")


class MyLocust(HttpLocust):
    task_set = NoSlowQTaskSet
    host = "http://localhost:9528"
```
我们把master跑起来，启动两个slaves。在没有启动压测前，我们浏览器访问一下

```
http://127.0.0.1:8089/export/prometheus
```
返回结果如下：
![]({{ site.url }}/assets/locust/exporter_empty.jpg)

这是使用prometheus_client库默认产生的信息，对我们数据采集没有影响，如果想关注master进程可以在grafana上创建相应的监控大盘。

接着我们启动10个并发用户开始压测，继续访问下上面的地址：
![]({{ site.url }}/assets/locust/exporter_start.jpg)

可以看到，locust_stats_avg_content_length、locust_stats_current_rps等信息都采集到了。


## Prometheus部署
exporter已经ready了，接下来就是把prometheus部署起来，拉取metric数据了。
1) 准备好了docker环境，我们直接把prometheus镜像拉下来：

```shell
docker pull prom/prometheus
```

2) 接下来我们创建一个yml配置文件，准备覆盖到容器中的/etc/prometheus/prometheus.yml

```
global:
  scrape_interval:     10s
  evaluation_interval: 10s
 
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus
          
  - job_name: locust
    
    metrics_path: '/export/prometheus'
    static_configs:
      - targets: ['192.168.31.112:8089']
        labels:
          instance: locust
```

3) 启动prometheus，将9090端口映射出来，执行命令如下：

```
docker run -itd -p 9090:9090 -v ~/opt/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

接下来我们访问Prometheus的graph页面，查询下是否有数据了

```
http://127.0.0.1:9090/graph
```
![]({{ site.url }}/assets/locust/prom_graph.jpg)

OK，有数据，太棒了！

## Grafana部署和配置
1）首先我们需要下载grafana的镜像：

```
docker pull grafana/grafana
```

2) 启动一个grafana容器,将3000端口映射出来：

```
docker run -d -p 3000:3000 grafana/grafana
```

3）网页端访问locahost:3000验证部署成功
![]({{ site.url }}/assets/locust/grafana_home.jpg)

4) 选择添加prometheus数据源
![]({{ site.url }}/assets/locust/add_datasource.jpg)
![]({{ site.url }}/assets/locust/add_db_succ.jpg)
![]({{ site.url }}/assets/locust/db_list.jpg)

5) 导入模板
导入模板有几种方式，这里我们使用导入[json](https://gist.github.com/bugVanisher/042759e1d6971e14bc21f1d0510688f2)
![]({{ site.url }}/assets/locust/import.jpg)
![]({{ site.url }}/assets/locust/import_json.jpg)


## 效果展示
经过一系列『折腾』之后，是时候看看效果了。使用 Docker + Locust + Prometheus + Grafana 到底可以搭建怎样的性能监控平台呢？相比较 Locust 自带的监控平台，我们搭建的性能监控平台究竟有什么优势呢？接下来就是展示成果的时候啦！

![]({{ site.url }}/assets/locust/dashboard.jpg)

这个监控方案不仅提供了炫酷好看的图表，，还能持久化存储所有压测数据，可以使用Share Dashboard功能保存测试报告并分享，简直太方便！

![]({{ site.url }}/assets/locust/share.jpg)


之前Locust一直被人诟病，主要有两方面，一是监控平台做得太过于简陋，二是CPython的GIL导致需要起更多的slave来充分利用多核CPU且并发大之后response time不稳定。对于第一个问题，相信读到篇文章的你应该认为这不是什么问题了，而第二点，如果我们无法摆脱GIL，那自己实现一个slave端呢？后面我们一起来学习下，如何实现一个Locust的slave端，敬请期待~