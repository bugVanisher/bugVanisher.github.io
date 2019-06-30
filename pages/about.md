---
layout: page
title: About
description: 快乐工作，认真生活！
keywords: bugVanisher, 见欢
comments: true
menu: 关于
permalink: /about/
---
## 关于我
我是见欢。

一名测试开发。

## 我所理解的测试

做测试最高境界是什么？

我比较认同的答案是：帮助自己所在的团队树立正确的质量观念；帮助所在的团队建立起有效预防和发现 Bug 的流程体系与技术栈。

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## 技能关键字

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
