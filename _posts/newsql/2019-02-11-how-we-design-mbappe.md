---
layout: post
title: "慢查询告警系统(测试阶段)——系统设计"
author: "见欢"
---
### 业务背景和目标
**业务背景**、**业务目标**请参考 [慢查询告警系统(测试阶段)——需求分析]({% post_url 2019-02-09-why-we-need-sql-analyse %}) 
### 参考文档
  无
### 功能需求用例

| 序号 | 应用 | 需求用例 |
| 1 | 客户端 | 框架注入 |
| 2 | worker | 数据库实例信息获取 |
| 3 | worker | Sql语句、参数的获取<br>生成时间<br>触发入口 |

### 容量需求
  根据实际情况预估
### 性能需求
  采用异步消息，消费tps预估不高于2000tps
### 可用性需求
  99.9%
### 扩展性需求
  后续功能可以按以下几方面扩展:
  1、按表订阅告警
  2、
### 安全需求
  不同的业务需要进行数据隔离，即不同的角色不能看到对方的SQL等内容
### 系统现状
  无
### 系统边界分析设计
![]({{ site.url }}/assets/newsql/系统架构图.png) 

系统说明：  
agent：作为信息拦截，采集然后发送异步消息的客户端  
worker：消息消费，SQL分析，入库，告警  
web：订阅管理、SQL处理、查询等

### 核心业务流程设计
sql实体状态图
![]({{ site.url }}/assets/newsql/sql实体状态图.png) 
### 数据分析设计
sql实体相关表设计
![]({{ site.url }}/assets/newsql/数据库设计.png) 

通知模块相关设计  

待补充...

### 关键用例设计
sql的数量非常多，因此需要对sql进行了参数化(模板化)处理，如下例子：
{% highlight sql %}
#Input
select * from category where game_id in (2,13,13,13,16,26,28,228,228,759);
#Output
select * from category where game_id in (?);

#Input
select server_id, name from game_server where pub_name="myname" and state=1 and is_common_server=2 and is_top=1 or (name in ("crossfire", "new")) group by name order by grade desc;
#Output
select server_id, name from game_server where pub_name=? and state=? and is_common_server=? and is_top=? or (name in (?)) group by name order by grade desc;

#Input
select * from shop where shop_name like '%ucdamai%' order by ctime desc limit 0,15;
#Output
select * from shop where shop_name like ? order by ctime desc limit ?,?;

{% endhighlight %}

同一个app_name下同一个执行数据库只有唯一的一条参数化 sql，保证了数据量可控，人工易于处理；参数化sql关联实际的sql（含多条），前端查询时由参数化sql获取实际的sql。

1、新增sql判断
![]({{ site.url }}/assets/newsql/新增SQL判断.png) 
