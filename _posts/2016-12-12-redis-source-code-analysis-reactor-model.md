---
layout: post
title: "redis 中的 reactor 模型分析"
date: 2016-12-12 23:30:00
tags: redis reactor
---

redis 中的 reactor 模型分析

在学习 redis 源码时，发现redis使用的是 reactor 模式的事件驱动方式，为了进一步学习 redis 和 reactor 模式，又不想重复造轮子，干脆动手将 redis 中的 reactor 模式的框子娄出来，做了一个小 demo，废话不多说，源码可以通过下面的网址下载。

> https://github.com/small-cat/my_redis

参考文章： <br>
http://www.open-open.com/lib/view/open1452486726589.html