---
layout: page
title: About
description: 快乐工作，认真生活！
keywords: Gannicus-yu, 见欢
comments: true
menu: 关于
permalink: /about/
---

我是见欢。

一名测试开发。

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
