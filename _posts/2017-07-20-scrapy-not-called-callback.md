---
layout: post
title: "scrapy - Request函数中回调函数未被调用的问题"
date: 2017-07-19 00:10:00
tags: scrapy callback
---
本文主要是讲述在 scrapy 爬虫时，发现 Request 函数中的回调函数没有被调用的解决办法

在 scrapy 中，

    scrapy.Request(url, headers=self.header, callback=self.parse_detail)

调试的时候，发现回调函数 `parse_detail` 没有被调用，这可能就是被过滤掉了，查看 scrapy 的输出日志 `offsite/filtered` 会显示过滤的数目。这个问题如何解决呢，查看手册发现

	https://doc.scrapy.org/en/latest/faq.html?highlight=offsite%2Ffiltered

这个问题，这些日志信息都是由 scrapy 中的一个 middleware 抛出的，如果没有自定义，那么这个 middleware 就是默认的 `Offsite Spider Middleware`，它的目的就是过滤掉哪些不在 `allowed_domains` 列表中的请求 requests。

再次查看手册中关于 `OffsiteMiddleware` 的部分

	https://doc.scrapy.org/en/latest/topics/spider-middleware.html#scrapy.spidermiddlewares.offsite.OffsiteMiddleware

两种方法能够使 requests 不被过滤: <br>
1. 在 `allowed_domains` 中加入 url <br>
2. 在 scrapy.Request() 函数中将参数 `dont_filter=True` 设置为 True <br>

如下摘自手册
    
    If the spider doesn’t define an allowed_domains attribute, or the attribute is empty, the offsite middleware will allow all requests.
    
    If the request has the dont_filter attribute set, the offsite middleware will allow the request even if its domain is not listed in allowed domains.
