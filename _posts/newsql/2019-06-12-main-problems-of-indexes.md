---
layout: post
title: "数据库索引常见问题"
author: "见欢"
categories: 慢查询分析系统
description: 数据库索引的常见问题
keywords: 慢查询,索引,问题
---

## 前言
这几年的工作中遇到非常多与数据库相关的问题，索引是其中最常见，也是最容易出现问题的，下面我结合自己的学习和工作中遇到的索引相关问题进行了一些总结。


为了更好地讨论sql语句索引相关问题，在开始之前，我们先假设有下面这么一张商品表，然后根据业务情况，讨论它对应的查询语句和索引建立情况。

```sql
CREATE TABLE `goods_item` (
  `goods_id` bigint(20) NOT NULL COMMENT '商品ID',
  `seller_id` bigint(20) NOT NULL COMMENT '卖家ID',
  `category_id` bigint(20) NOT NULL COMMENT '商品分类ID',
  `title` varchar(20) NOT NULL COMMENT '用户输入标题',
  `price` double(10,3) NOT NULL COMMENT '售价',
  `account_id` varchar(50) DEFAULT '' COMMENT '交易帐号ID',
  `goods_status` smallint(1) NOT NULL COMMENT '1-公司审核2-待运营审核3-已上架4-用户下架5-库存不足6-运营下架',
  `audit_status` smallint(1) NOT NULL COMMENT '审核状态',
  `game_id` int(5) NOT NULL COMMENT '游戏id',
  `publisher_id` int(10) NOT NULL DEFAULT '0' COMMENT '游戏发行商ID',
  `domain_id` int(10) DEFAULT '0' COMMENT '服务区ID',
  `server_id` int(10) NOT NULL DEFAULT '0' COMMENT '服务器ID',
  `storage` int(10) NOT NULL COMMENT '库存量',
  `ctime` int(13) NOT NULL COMMENT '创建时间',
  `utime` int(13) NOT NULL COMMENT '更新时间',
  `reason` varchar(50) DEFAULT '' COMMENT '下架原因',
  `p_id` int(2) DEFAULT NULL COMMENT '商品分类',
  PRIMARY KEY (`goods_id`),
  KEY `idx_gameId_goodsState_price` (`game_id`,`goods_status`,`price`) USING BTREE,
  KEY `idx_ctime` (`ctime`) USING BTREE,
  KEY `idx_seller_status_ctime` (`seller_id`, `goods_status`, `ctime`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 comment='商品信息表';
```

<a name="bok9k"></a>
## 一、无法命中任何索引
MySQL查询优化器对索引的选择，其中一个非常重要的原则是——**最左前缀原则**。这里的最左前缀原则是由于索引采用的是B+树的数据结构存储决定的。被索引的列或多个列，按照一个合理的深度分部在一颗B+树上。这个与组合索引的顺序有关，而与where条件的前后顺序无关。举个例子，如果我们想要获取所有上架中的商品并按发布时间倒序，如下：

```sql
select * from goods_item where goods_status=2 and price<100 order by ctime desc;
```

显然以上SQL根本无法使用索引。如果需要查询某个卖家的所有上架商品，我们会这样查，它们对数据库而言是一样的，这些查询语句符合最左前缀原则，能够正常使用索引：

```sql
select * from goods_item where goods_status=2 and seller_id=123;
# 等价于
select * from goods_item where seller_id=123 and goods_status=2;
```


<a name="gHy1E"></a>
## 二、有索引但不走索引
<a name="ai4ZM"></a>
### 数据原因

- 符合索引规则，但结果数据量基本为全量数据

假设我们的商品最终状态都是3-下架，数据库中存在500w数据，其中495w的商品处于下架状态，而只有在途的5w商品处于非下架状态，那么MySQL的查询优化器很可能是不会选择以goods_status为最左前缀的索引的。然而，如果业务条件只需要查询非上架状态的商品，则索引能发挥正常作用。

- 数据量少的时候，不走索引

当表数据量很少的时候，MySQL的查询优化器会选择全表扫描，因为这样的成本也很小。这个在测试环境的时候最明显，测试环境明明看到explain结果的type为all，线上却能命中索引。

<a name="opCIN"></a>
### 表结构不合理

- 索引字段 is null 不走索引

如果字段A允许为空，且创建了该字段的单列索引，如果需要查询所有字段且where字句包含A is null则无法使用此索引(查询count(*)可以命中索引)。但包含A的组合索引还是可以用到对应的索引；

- 单键值的b+树索引列上存在null值，导致COUNT(*)不能走索引

<a name="8hNLT"></a>
### where条件限制

- 没有查询条件

这个就不用说了，没有条件，无法使用索引啊，除了一些小表，必须要添加查询条件。

- 隐式转换导致不走索引

如果我们查询某个游戏账号123，如下：

```sql
select * from goods_item where account_id = 123;
```
这样的语句是不会走索引的，因为这个过程做了隐式的类型转换，正确的写法应该为：

```sql
select * from goods_item where account_id = "123";
```

- !=或者<>(不等于），可能导致不走索引
- like '%keyword' 百分号在前不走索引
- not in ,not exist 
- 索引列上有函数运算，导致不走索引

在where字句中使用函数运算需要特别小心，因为对于查询过程，每一条记录都需要计算，数据库就不会走索引了。

```sql
select * from goods_item where price/100 > 1;
```



<a name="Xpbef"></a>
## 三、命中索引但仍然很慢
以下问题都基于假设商品信息表有上千万的数据：

- 过渡翻页

假如goods_item表有1000w数据，排序后，需要翻到最后几页，这样的场景非常容易造成慢查询，可以通过join分页和子查询分页优化；当然更暴力的做法，比如前台商品列表的翻页功能，直接控制翻页的最大数量？

- 索引效率低下

如果查询价格区间商品，使用utime排序，那么会出现filesort,会产生额外的性能损耗。

- 索引区分度低

如果商品p_id字段的值只有两种，那么使用p_id作为最左前缀，索引效率将会非常低

- 查询范围过大（区间查询或者过大的limit）

由于某些场景，需要查询近半年的商品，或许近半年的商品可能就有500w，那么一查可能都是慢查询

- 索引过多

如果不接搜索引擎，完全由数据库承接查询功能，那么需要建立非常多的索引，有两方面影响：

-- 1.索引过多会影响mysql的执行计划，实际走的索引并非与你期望的一样。

-- 2.索引过多会影响写入性能。

- 数据量过大

数据量越大，查询的响应时间就会相应增加。

