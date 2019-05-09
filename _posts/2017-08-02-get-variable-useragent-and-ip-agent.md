---
layout: post
title: "获取变化的的User-Agent和IP代理"
date: 2017-08-02 23:30:00
tags: user-agent 反爬虫 代理
---
获取变化的User-Agent和IP代理，避免反爬虫策略

## 如何获取不断变化的 User-Agent
> User Agent中文名为用户代理，简称 UA，它是一个特殊字符串头，使得服务器能够识别客户使用的操作系统及版本、CPU 类型、浏览器及版本、浏览器渲染引擎、浏览器语言、浏览器插件等。

比如，firefox 浏览器中的 user agent 可能为 `Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:54.0) Gecko/20100101 Firefox/54.0`，其他浏览器的 user agent 都是不一样的。那么如何随机更换 user agent 呢。

如果我们能获取到所有可能出现的 user agent，那么我们只需要随机从这些 user agent 列表中取一条就可以了。正好，有人已经帮我们实现了这个事情。在 github 上搜索 useragent，选择 fake-useragent，作者维护了一个可变的user agent 列表，地址为 `https://fake-useragent.herokuapp.com/browsers/0.1.7`，这个地址是可以根据版本变化的，以前版本的链接可能已经无法访问。这里列出了所有作者维护的 user-agent(感谢作者和开源的贡献，给我们提供的方便)。那么，我们通过获取到这些 user-agent，然后随机的从这个列表中取 user-agent 就可以。

这当然是一种方法，但既然已经有了 fake-useragent，那么我们就不需要这么做了。大家也可以看帮助文档中的使用方法。

    # 安装 fake-useragent
    pip install fake-useragent
    
使用 fake-useragent

    from fake-useragent import UserAgent
    ua = UserAgent()
    ua.ie   # 输出 ie 中的一个随机的 useragent
    ua.firefox # 输出 firefox 中的一个随机的 useragent
    # 可以查看帮助文档
    
## 在 scrapy 中获取变化的 useragent
在 scrapy 爬虫的时候，比如爬取知乎的文章，知乎有一定的反爬虫限制，会检测你的 useragent，如果访问量很大，就会暂时停止你访问服务器，那么我们就需要不断获取变化的 useragent 来避免被知乎禁止访问服务器。

大家在 scrapy 官方帮助文档的时候，应该可以看到 scrapy 的系统设计图 <br>
![scrapy architecture](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/scrapy_architecture.png)

当我们通过 spider yield 一个 requests 的时候，首先通过 spider middlewares 到达 scrapy engine，然后 engine 将 requests 放到 scheduler 的队列中，通过 scheduler 调度队列中的 requests ，scheduler 选中一个 requests 后，将 requests 通过 engine 传递给 downloader，在这之前，必然会经过 downloader middlewares，downloader 下载好之后，将 response 返回给 engine，engine 在将 response 返回给 spider，我们就可以在 spider 中调用 callback 进行解析，简单的流程大概就是这样。

那么，我们在将 requests 提交给 downloader 进行下载之前，就需要将 user-agent 进行变化，也就是每次都需要随机取一个 user-agent 才能提交到 downloader 进行下载，否则就可能被知乎禁掉。在提交到 downloader 的时候，必然会经过 downloader middlewares，所以我们实现随机获取 user-agent 的逻辑部分，可以在 downloader midllewares 这里实现。

    class RandomUserAgentMiddleWare(object):
        """
        随机更换User-Agent
        """
        def __init__(self,crawler):
            super(RandomUserAgentMiddleWare, self).__init__()
            self.ua = UserAgent()
            self.ua_type = crawler.settings.get("RANDOM_UA_TYPE", "random")
    
        @classmethod
        def from_crawler(cls, crawler):
            return cls(crawler)
    
        def process_request(self, request, spider):
            def get_ua_type():
                return getattr(self.ua, self.ua_type)   # 取对象 ua 的 ua_type 的这个属性, 相当于 self.ua.self.ua_type
    
            # random_useragent = get_ua_type()
            request.headers.setdefault('User-Agent', get_ua_type())
    
完成 downloader middleware 的时候，我们需要像在 pipelines 一样，在 settings.py 中进行设置，让我们的 middleware 生效。在 settings.py 这个文件中，有一个 `DOWNLOADER_MIDDLEWARES` 这个字典对象，一般默认是注释的，打开注释，并将默认的 middleware 注释掉，按照规则加上我们自己的 middleware。字典后面键对应的值，越大表示被调用的优先级越低，调用的顺序越靠后，如果不希望注释掉默认的或者其他的 middleware，就要将我们自己实现的这个 middleware 的值设置的最大，这样，就能保证我们自己实现的 middleware 是在最后一个被调用，user-agent 不会被其他的影响。

当然，上面的代码中，设置了一个 `RANDOM_UA_TYPE`，这样，就能通过设置这个值，来获取不同浏览器的 user-agent

    RANDOM_UA_TYPE = "safari"
    
## 在 scrapy 中设置代理
在 scrapy 中设置代理是非常简单的，比如上面的代码中，在 `process_request` 函数的最后加上一句

    request.meta["proxy"] = "ip:port"
    
上面就设置了 scrapy 的代理了，非常简单。但是，如何获取这些代理的 ip 和 port 是一个问题。

我们发现，西刺网为我们提供了很多可用的代理服务器ip和端口，我们只需要将这个网站上面的所有的需要的代理数据获取下来，放在文件或者数据库中，每次随机取一个使用，能够实现通过随机代理的方式爬取网站数据。如何爬取所有的代理数据，大家可以参考我之前的一篇博文[ scrapy简单入门 - 爬取伯乐在线所有文章 ]。
