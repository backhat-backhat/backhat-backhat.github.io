---
layout: post
title: "scrapy的简单入门-爬取伯乐在线所有文章"
date: 2017-07-09 23:30:00
tags: scrapy 爬虫
---
本文介绍了 scrapy 的简单入门，使用 scrapy 爬取网站所有文章，文章结构如下: <br>
1. 分析网页结构 <br> 
2. 使用 css selector 的方法提取元素<br> 
3. 开始 scrapy 工程 <br>
4. 获取所有文章url，爬取文章数据 <br>
5. 下载图片 <br>
6. 使用 item 和 itemloader <br>
7. 将数据导出到 json 格式的文件中 <br>
8. 将数据保存到 mysql 数据库中

scrapy 是一个用 python 语言编写的，为了爬取网站数据，提取结构性数据而编写的应用框架。
# 环境
本文使用的环境：<br>
python 3.5.2 <br>
pip 9.0.1 <br>
操作系统： Ubuntu 16.04
## pythton 环境搭建
在官网下载 Ubuntu 环境下的python3.5的安装包，安装，安装完成后，检查一下 python 的安装情况，一般pyhton安装的时候，pip 也是一起安装好的，如果没有安装完全，再将 pip 也一起安装好。
## 虚拟环境搭建
现在Ubuntu默认是安装 python2.7 的，避免两个环境之间切换的麻烦，我们安装 python 虚拟环境来解决这个问题。<br>

    pip install virtualenv
    pip install virtualwrapper
    pip list # 查看已安装
virtualenv 能够通过根据不同的 python 版本创建对应不同版本的虚拟环境，virtualwrapper 能够方便的在不同的虚拟环境之间进行切换。安装完成之后，下面我们创建一个 python3.5.2 版本的虚拟环境

    source /usr/local/bin/virtualwrapper.sh #这个与 windows 不一样，需要先执行一下脚本才能生效，大家可以打开这个文件看一下
    # 创建一个名为 py3Scrapy 的虚拟环境
    mkvirtualenv py3Scrapy -p /usr/bin/python3.5
    # workon 查看创建好的虚拟环境，虚拟环境的保存路径可以通过 `VIRTUALENV_PYTHON` 来配置
    workon
    workon py3Scrapy # 进入选择的虚拟环境

如下图所示： <br>
![进入虚拟环境](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/1.png)

python的版本也能查看得到，进入虚拟环境之后，在shell前面会出现虚拟环境的名称，退出虚拟环境

    deactivate

好了，创建好环境之后，现在来开始我们的 scrapy 之旅吧。

## scrapy 环境搭建
scrapy 是基于 twisted 框架的，大家会发现，安装 scrapy 的时候，会需要安装很多包。

    pip install scrapy

使用 pip 进行安装，方便，但是这种默认的安装方式，实在官网下载安装包来进行安装的，比较慢，大家可以**使用豆瓣源来进行安装**

    pip install scrapy -i https://pypi.douban.com/simple

这种方式，下载会非常的快，安装其他的包都可以使用这种方法，但是，如果使用豆瓣源安装的时候，提示找不到符合版本的安装包，那就使用第一种办法进行下载安装，因为豆瓣源可能没有官网那么及早更新。

因为每个人的环境都可能存在差异，安装过程中会出现一些问题。当如果报错，twisted 安装失败的时候，建议从官网下载 twisted 安装包，自行进行安装，安装完成之后，再接着继续上面 scrapy 的安装，安装完成之后，检查一些安装结果

    scrapy -h

# 使用 scrapy 获取某一个文章的信息
好了，环境准备好之后，接下来我们来分析一下伯乐在线的文章网页结构
## 分析伯乐在线某一篇文章的网页结构和url
伯乐在线网站不需要我们先登录，然后才能访问其中的内容，所以不需要先模拟登录，直接就能访问网页。伯乐在线地址为 `https://www.jobbole.com`，这上面的文章质量还是不错的，大家有时间可以看看。

我们随便找一篇文章试图来分析一下，比如 `http://blog.jobbole.com/111469/`，F12进入浏览器调试窗口，从全文分析，比如我们想获取文章标题，文章内容，文章创建时间，点赞数，评论数，收藏数，文章所属类别标签，文章出处等信息 <br>
## 使用 scrapy shell 的方法获取解析网页数据
打开文章链接，我们获取到的是一个html页面，那么如何获取上面所说的那些数据呢，本文通过 CSS 选择器来获取(不了解 CSS selector的小伙伴可以先去熟悉一下 `http://www.w3school.com.cn/cssref/css_selectors.asp`)。 scrape 为我们提供了一个 shell 的环境，可以方便我们进行调试和实验，验证我们的css 表达式能够成功获取所需要的值。下面启动 scrapy shell

    scrapy shell "http://blog.jobbole.com/111469/"

scrapy 将会帮助我们将`http://blog.jobbole.com/111469/`这个链接的数据捕获，现在来获取一下文章标题，在浏览器中找到文章标题，`inspect element` 审查元素，如下图所示：<br>
![css get title](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/2.png)<br>
文章标题为**王垠：如何掌握所有的程序语言**，从上图获知，这个位于一个 class 名为 `entry-header` 的 div 标签下的子标签 h1 中，那我们在 scrapy shell 通过 css 选择器来获取一下，如下图所示：<br>
![get title](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/3.png)<br>
仔细查看上图，注意一些细节。通过 response.css 方法，返回的结果是一个 selector，不是字符串，在这个 selector 的基础上可以继续使用 css 选择器。通过 extract() 函数获取提取的标题内容，返回结果是一个 list，注意，这里是一个 list ，仍然不是字符串 str，使用 extract()[0] 返回列表中的第一个元素，即我们需要的标题。

但是，如果标题没有获取到，或者选择器返回的结果为空的话，使用 extract()[0] 就会出错，因为试图对一个空链表进行访问，这里使用 extract_first() 方法更加合适，可是使用一个默认值，当返回结果为空的时候，返回这个默认值

    extract_first("")   # 默认值为 ""

此处仅仅是将 title 标题作为一个例子进行说明，其他的就不详细进行解释了，主要代码如下所示：

{% highlight ruby %}
        title = response.css(".entry-header h1::text").extract()[0]
        match_date = re.match("([0-9/]*).*",
                              response.css(".entry-meta-hide-on-mobile::text").extract()[0].strip())
        if match_date:
            create_date = match_date.group(1)

        votes_css = response.css(".vote-post-up h10::text").extract_first()
        if votes_css:
            vote_nums = int(votes_css)
        else:
            vote_nums = 0

        ma_fav_css = re.match(".*?(\d+).*",
                              response.css(".bookmark-btn::text").extract_first())
        if ma_fav_css:
            fav_nums = int(ma_fav_css.group(1))
        else:
            fav_nums = 0

        ma_comments_css = re.match(".*?(\d+).*",
                                   response.css("a[href='#article-comment'] span::text").extract_first())
        if ma_comments_css:
            comment_nums = int(ma_comments_css.group(1))
        else:
            comment_nums = 0

        tag_lists_css = response.css(".entry-meta-hide-on-mobile a::text").extract()
        tag_lists_css = [ele for ele in tag_lists_css if not ele.strip().endswith('评论')]
        tags = ','.join(tag_lists_css)

        content = response.css(".entry *::text").extract()
{% endhighlight %}

解释一下 create_date，通过获取到的值，存在其他非时间的数据，通过 re.match 使用正则表达式来提去时间。

好了，所有需要的值都提取成功后，下面通过 scrapy 框架来创建我们的爬虫项目。
## 创建爬虫项目
开始我们的爬虫项目

    scrapy startproject ArticleSpider

scrapy 会为我们创建一个名为 ArticleSpider 的项目 <br>
进入到 ArticleSpider 目录，使用基本模板创建

    scrapy genspider jobbole blog.jobbole.com

创建完成之后，我们使用 pycharm 这个IDE打开我们创建的爬虫项目，目录结构如下所示： <br>

    ├── ArticleSpider
    │   ├── items.py
    │   ├── middlewares.py
    │   ├── pipelines.py
    │   ├── __pycache__
    │   ├── settings.py
    │   ├── spiders
    │   │   ├── __init__.py
    │   │   ├── jobbole.py
    │   │   └── __pycache__
    └── scrapy.cfg

我们可以在 items.py 里面定义数据保存的格式，在 middlewares.py 定义中间件，在 piplines.py 里面处理数据，保存到文件或者数据库中等。在 jobbole.py 中对爬取的页面进行解析。

下面，我们首先需要做的，就是利用我们编写的 css 表达式，获取我们提取的文章的值。在 jobbole.py 中，我们看到

{% highlight ruby %}
    class JobboleSpider(scrapy.Spider):
        name = 'jobbole'
        allowed_domains = ['blog.jobbole.com']
        start_urls = ['http://blog.jobbole.com/all-posts/']
    
        def parse(self, response):
        pass
{% endhighlight %}

scrapy 为我们创建了一个 JobboleSpider 的类，name 是爬虫项目的名称，同时定义了域名以及爬取的入口链接。scrapy 初始化的时候，会初始化 `start_urls` 入口链接列表，然后通过 `start_requests` 返回 Request 对象进行下载，调用 parse 回调函数对页面进行解析，提取需要的值，返回 item。

所以，我们需要做的，就是将我们在上一小节编写的代码放在 parse 函数中，同时，将 start_urls 的值，改为上面我们在 scrapy shell 爬取的页面的地址`http://blog.jobbole.com/111469/`，因为我们这里还没有讲到通过 item 获取我们提取的值，此处你可以通过 print() 函数将值进行打印。在 shell 中启动爬虫（先进入我们的工程目录）

    scrapy crawl jobbole

既然我们使用的了 pycharm 这个IDE，那么我们就不用 shell 来启动爬虫，在 ArticleSpider 目录下创建一个 main.py 文件

    from scrapy.cmdline import execute
    import sys
    import os
    
    sys.path.append(os.path.dirname(os.path.abspath(__file__)))
    execute(["scrapy", "crawl", "jobbole"])

上面的代码，就是将当前项目路径加入到 path 中，然后通过调用scrapy 命令行来启动我们的工程。然后，通过设置断点调试，一步一步查看我们的提取的变量的值是否正确。

> 注意：启动之前，将 settings.py 中的 `ROBOTSTXT_OBEY` 这个参数设置为 False

这样，我们就爬取到了伯乐在线的这一篇文章了。
# 扩展，爬取所有的文章
既然我们已经能够获取到某一篇文章的数据，那么下面就来获取所有文章的链接。
## 扩展一：获取所有 url 链接
伯乐在线所有文章链接的入口地址为 `http://blog.jobbole.com/all-posts/`，通过浏览器进入调试模式查看文章列表的链接，如下图所示<br>
![文章列表链接](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/4.png)<br>
文章链接是在 id 为 archive 的 div 标签下的子 div 标签之下， class 为 post-thumb，这个下面的子标签 a 的 href 属性，仍使用上面说的 scrapy shell 的方法，如下图所示 <br>
![url list](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/5.png)<br>
可以看出，获得了当前页面所有的文章的 url，这仅仅是当前页面的所有 url，我们还需要获取下一页的 url，然后通过下一页的 url 进入到下一页，获取下一页的所有文章的 url，依次类推，知道爬取完所有的文章 url。

在文章列表的最后，有翻页，分析如下<br>
![next url](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/6.png)<br>
下一页是 class 为 next page-numbers 的 a 标签中，如下图 <br>
![get next url](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/7.png)<br>
既然现在所有的 url 都能够获取到了，那么现在我们将 jobbole.py 中的 parse 函数修改一下

{% highlight ruby %}
    def parse(self, response):
        post_nodes = response.css("#archive .floated-thumb .post-thumb")    # a selector, 可以在这个基础上继续做 selector
        
        for post_node in post_nodes:
            post_url = post_node.css("a::attr(href)").extract_first("")
            yield Request(url=parse.urljoin(response.url, post_url),
                          callback=self.parse_detail)

        # 必须考虑到有前一页，当前页和下一页链接的影响，使用如下所示的方法
        next_url = response.css("span.page-numbers.current+a::attr(href)").extract_first("")
         if next_url:
             yield Request(url=parse.urljoin(response.url, next_url), callback=self.parse)

    def parse_detail(self, response):
    """作为回调函数，在上面调用"""
    title = response.css(".entry-header h1::text").extract()[0]
        match_date = re.match("([0-9/]*).*",
                              response.css(".entry-meta-hide-on-mobile::text").extract()[0].strip())
        if match_date:
            create_date = match_date.group(1)

        votes_css = response.css(".vote-post-up h10::text").extract_first()
        if votes_css:
            vote_nums = int(votes_css)
        else:
            vote_nums = 0

        ma_fav_css = re.match(".*?(\d+).*",
                              response.css(".bookmark-btn::text").extract_first())
        if ma_fav_css:
            fav_nums = int(ma_fav_css.group(1))
        else:
            fav_nums = 0

        ma_comments_css = re.match(".*?(\d+).*",
                                   response.css("a[href='#article-comment'] span::text").extract_first())
        if ma_comments_css:
            comment_nums = int(ma_comments_css.group(1))
        else:
            comment_nums = 0

        tag_lists_css = response.css(".entry-meta-hide-on-mobile a::text").extract()
        tag_lists_css = [ele for ele in tag_lists_css if not ele.strip().endswith('评论')]
        tags = ','.join(tag_lists_css)

        # cpyrights = response.css(".copyright-area").extract()
        content = response.css(".entry *::text").extract()
{% endhighlight %}
		
1. 获取文章列表页中的文章url，交给 scrapy 下载后并进行解析，即调用 parse 函数解析
2. 然后获取下一页的文章 url，按照1 2 循环

对于 parse 函数，一般做三种事情 <br>
a. 解析返回的数据 response data <br>
b. 提取数据，生成 ITEM <br>
c. 生成需要进一步处理 URL 的 Request 对象 <br>

> 某些网站中，url 仅仅只是一个后缀，需要将当前页面的url+后缀进行拼接，使用的是 `parse.urljoin(base, url)`，如果 urljoin 中的 url 没有域名，将使用base进行拼接，如果有域名，将不会进行拼接,此函数在 python3 的 urllib 库中。Request(meta参数)：meta参数是一个字典{},作为回调函数的参数

这样，我们就获得了所有的文章
## 扩展二：使用item，并保存图片到本地
上一小节提到了， parse 函数提取数据之后，生成 item，scrapy 会通过 http 将 item 传到 pipeline 进行处理，那么这一小节，我们使用 item 来接收 parse 提取的数据。在 items.py 文件中，定义一个我们自己的数据类 JobBoleArticleItem，并继承 scrapy.item 类

{% highlight ruby %}
    class JobBoleArticleItem(scrapy.Item):
        title = scrapy.Field()          # Field()能够接收和传递任何类型的值,类似于字典的形式
        create_date = scrapy.Field()    # 创建时间
        url = scrapy.Field()            # 文章路径
        front_img_url_download = scrapy.Field()
        fav_nums = scrapy.Field()       # 收藏数
        comment_nums = scrapy.Field()   # 评论数
        vote_nums = scrapy.Field()      # 点赞数
        tags = scrapy.Field()           # 标签分类 label
        content = scrapy.Field()        # 文章内容
        object_id = scrapy.Field()      # 文章内容的md5的哈希值，能够将长度不定的 url 转换成定长的序列
{% endhighlight %}

Field() 对象，能够接收和传递任何类型的值，看源代码，就能发现，Field() 类继承自 dict 对象，具有字典的所有属性。

注意，在上面定义的类中，我们增加了一个新的成员变量 `front_img_url_download`，这是保存的是文章列表中，每一个文章的图片链接。我们需要将这个图片下载到本地环境中。既然使用了 item 接收我们提取的数据，那么 parse 函数就需要做相应的改动

{% highlight ruby %}
    def parse(self, response):
        post_nodes = response.css("#archive .floated-thumb .post-thumb")    # a selector, 可以在这个基础上继续做 selector
        
        for post_node in post_nodes:
            post_url = post_node.css("a::attr(href)").extract_first("")
            img_url = post_node.css("a img::attr(src)").extract_first("")
            yield Request(url=parse.urljoin(response.url, post_url),
                          meta={"front-image-url":img_url}, callback=self.parse_detail)

        # 必须考虑到有前一页，当前页和下一页链接的影响，使用如下所示的方法
        next_url = response.css("span.page-numbers.current+a::attr(href)").extract_first("")
         if next_url:
             yield Request(url=parse.urljoin(response.url, next_url), callback=self.parse)
{% endhighlight %}

同时，解析函数 parse_detail 也需要修改，将数据保存到我们的item中，只需要添加下面的部分就可

{% highlight ruby %}
        front_img_url = response.meta.get("front-img-url", "")
        article_item = JobBoleArticleItem() # 实例化 item 对象
        # 赋值 item 对象
        article_item["title"] = title
        article_item["create_date"] = create_date
        article_item["url"] = response.url
        article_item["front_img_url_download"] = [front_img_url] # 这里传递的需要是列表的形式，否则后面保存图片的时候，会出现类型错误，必须是可迭代对象
        article_item["fav_nums"] = fav_nums
        article_item["comment_nums"] = comment_nums
        article_item["vote_nums"] = vote_nums
        article_item["tags"] = tags
        # article_item["cpyrights"] = cpyrights
        article_item["content"] = ''.join(content)      # 取出的 content 是一个 list ,存入数据库的时候，需要转换成字符串
        article_item["object_id"] = gen_md5(response.url)
        yield article_item
{% endhighlight %}

这里，parse 函数成功生成了我们定义的 item 对象，将数据传递到 pipeline。那么，图片链接已经获取到了，我们如下将图片下载下来呢。

> 解释一下上面代码中的 front-img-url，这个是在 parse 函数中作为参数 meta 传递给 Request() 函数，回调函数调用 parse_detail，返回的 response 对象中的 meta 成员，将包含这个元素， meta 就是一个字典， response.meta.get("front-img-url") 将获取到我们传递过来的图片url

scrapy 提供了一个 ImagesPipeline 类，可直接用于图片操作，只需要我们在 settings.py 文件中进行配置即可。

在 settings.py 中，有一个配置参数为 `ITEM_PIPELINE`，这其实就是一个字典，当需要用到 pipeline 时，就需要在这个字典中进行配置，字典中存在的， scrapy 才会使用。字典中的 key 就是 pipeline 的类名，后面的数字表示优先级，数字越小表示越先调用，越大越靠后。既然我们现在需要使用到 scrapy 提供的图片下载功能，那么需要在这个字典中配置 ImagesPipeline

    ITEM_PIPELINES = {
       'scrapy.pipelines.images.ImagesPipeline': 1,
    }

同时，还需要在 settings.py 中配置，item 中哪一个字段是图片 url，以及图片需要存放什么位置

    IMAGES_URLS_FIELD = "front_img_url_download"     # ITEM 中的图片 URL，用于下载
    PROJECT_IMAGE_PATH = os.path.abspath(os.path.dirname(__file__))   # 获取当前文件所在目录
    IMAGES_STORE = os.path.join(PROJECT_IMAGE_PATH, "images")         # 下载图片的保存位置

这些参数，可以在 ImagesPipeline 类的源代码中查看到

> **注意：**上面配置好后，上面的代码是在工程路径下面创建一个 images 的目录，用于保存图片，运行 main.py，可能会出现如下错误: no module named PIL，这是因为图片操作需要 pillow 库，只需要安装即可
`pip install pillow`，快速安装，就按照我上面说的豆瓣源的方法。<br>
还可能出现"ValueError: Missing scheme in request url: h"的错误，这是因为图片操作，要求 `front_img_url_download` 的值为 list 或者可以迭代的对象，所以我们在 parse 函数中给 item 赋值的时候， `front_img_url_download` 就是赋值的 list 对象

好了，这些注意了之后，应该能够下载图片了。
## 扩展三：使用 itemloader
相信大家已经发现，虽然使用了 item，但是使用 css selecotor，我们的 parse 函数显得很长，而且，当数据量越来越大之后，一大堆的 css 表达式是很难维护的。在加上正则表达式的提取，代码会显得很臃肿。这里，给大家推荐是用 itemloader。itemloader 可以看成是一个容器。

首先，在 items.py 中，我们需要定义一个继承自 ItemLoader 的类

{% highlight ruby %}
    class ArticleItemLoader(ItemLoader):
        """
        自定义 ItemLoader, 就相当于一个容器
        """
        # 这里表示，输出获取的 ArticleItemLoader 提取到的值，都是 list 中的第一个值
        # 如果有的默认不是取第一个值，就在 Field() 中进行修改
        default_output_processor = TakeFirst()
{% endhighlight %}

将默认输出函数定为 TakeFirst()，即去结果 list 中的第一个值，定义了 ItemLoader 类之后，需要修改 jobbole.py 中的 `parse_detail` 函数了，现在就不再直接使用 css selector 了，使用 itemloader 中的 css 进行数据提取，新的 `parse_detail` 如下所示：

{% highlight ruby %}
    def parse_detail(self, response):
        front_img_url = response.meta.get("front-image-url", "")
        item_loader = ArticleItemLoader(item=JobBoleArticleItem(), response=response)
        article_item_loader = JobBoleArticleItem()
        item_loader.add_css("title", ".entry-header h1::text")  # 通过 css 选择器获取值
        item_loader.add_value("url", response.url)
        item_loader.add_css("create_date", ".entry-meta-hide-on-mobile::text")
        item_loader.add_value("front_img_url_download", [front_img_url])
        item_loader.add_css("fav_nums", ".bookmark-btn::text")
        item_loader.add_css("comment_nums", "a[href='#article-comment'] span::text")
        item_loader.add_css("vote_nums", ".vote-post-up h10::text")
        item_loader.add_css("tags", ".entry-meta-hide-on-mobile a::text")
        item_loader.add_css("content", ".entry *::text")
        item_loader.add_value("object_id", gen_md5(response.url))
        # item_loader.add_xpath()
        # item_loader.add_value()
        article_item_loader = item_loader.load_item()
        yield article_item_loader
{% endhighlight %}

这样，数据提取就全部交给了 itemloader 来执行了。代码整体都简洁和工整了很多。ItemLoader 有三个方法用于提取数据，分别是 `add_css()`, `add_xpath()`, `add_value()`，前两个分别是 css 选择器和 xpath 选择器，如果是值，就直接使用 `add_value()` 即可。

最后 `load_item()` 函数，将根据上面提供的规则进行数据解析，每一个解析的值都是以 list 结果的形式呈现，同时，将结果赋值 item。

但是，大家应该已经发现，之前我们只接使用 css selector 提取数据的时候，对于某些数据，需要使用正则表达式进行匹配才能获取所需的值，这里什么都没做，仅仅是通过 itemloader 提取了数据而已。所以，我们还需要重新定义我们的 item 类，这些操作在 item 中进行处理。修改 items.py 中的 JobBoleArticleItem 类，具体如下：

{% highlight ruby %}
    class JobBoleArticleItem(scrapy.item):
        title = scrapy.Field()
        create_date = scrapy.Field(     # 创建时间
            input_processor = MapCompose(get_date),
            output_processor = Join("")
        )
        url = scrapy.Field()            # 文章路径
        front_img_url_download = scrapy.Field(    # 文章封面图片路径,用于下载，赋值时必须为数组形式
            # 默认 output_processor 是 TakeFirst()，这样返回的是一个字符串，不是 list，此处必须是 list
            # 修改 output_processor
            output_processor = MapCompose(return_value)
        )
        front_img_url = scrapy.Field()
        fav_nums = scrapy.Field(        # 收藏数
            input_processor=MapCompose(get_nums)
        )
        comment_nums = scrapy.Field(    # 评论数
            input_processor=MapCompose(get_nums)
        )
        vote_nums = scrapy.Field(       # 点赞数
            input_processor=MapCompose(get_nums)
        )
        tags = scrapy.Field(           # 标签分类 label
            # 本身就是一个list, 输出时，将 list 以 commas 逗号连接
            input_processor = MapCompose(remove_comment_tag),
            output_processor = Join(",")
        )
        content = scrapy.Field(        # 文章内容
            # content 我们不是取最后一个，是全部都要，所以不用 TakeFirst()
            output_processor=Join("")
        )
        object_id = scrapy.Field()      # 文章内容的md5的哈希值，能够将长度不定的 url 转换成定长的序列
{% endhighlight %}

`input_processor` 对传入的值进行预处理， `output_processor` 对处理后的值按照规则进行处理和提取，比如 `TakeFirst()` 就是对处理的结果去第一个值。

`input_processor = MapCompose(func1, func2, func3, ...)` 这行代码，说明的是， Item 传入的这个字段的值，将会分别调用 MapCompose 中的所有传入的方法进行逐个处理，这个方法也是可以是 lambda 的匿名函数。

因为上面定义 ArticleItemLoader 类的时候，使用了默认的 `default_output_processor`，如果不想使用默认的这个方法，就在 Field() 中，使用 `output_processor` 参数覆盖默认的方法，哪怕什么都不做，也不会使用默认方法获取数据了。对上面那些方法定义如下：

{% highlight ruby %}
    def get_nums(value):
        """
        通过正则表达式获取 评论数，点赞数和收藏数
        """
        re_match = re.match(".*?(\d+).*", value)
        if re_match:
            nums = (int)(re_match.group(1))
        else:
            nums = 0
    
        return nums
    
    
    def get_date(value):
        re_match = re.match("([0-9/]*).*?", value.strip())
        if re_match:
            create_date = re_match.group(1)
        else:
            create_date = ""
        return create_date
    
    def remove_comment_tag(value):
        """
        去掉 tag 中的 “评论” 标签
        """
        if "评论" in value:
            return ""
        else:
            return value
    
    
    def return_value(value):
        """
        do nothing, 只是为了覆盖 ItemLoader 中的 default_processor
        """
        return value
{% endhighlight %}

**千万注意：**这些方法，每一个最后，都必须有 return，否则程序到后面将获取不到这个字段的数据，再次访问这个字段的时候，就会报错。
## 扩展四：将数据导出到 json 文件中
好了，既然已经将数据通过 ItemLoader 获取到了，那么我们现在就将数据从 pipeline 输出到 json 文件中。

将数据以 json 格式输出，可以通过 json 库的方法，也可以使用 scrapy.exporters 的方法。
### json 库
我们已经知道，对数据的处理，scrapy 是在 pipeline 中进行的，所以，我们需要在 pipelines.py 中定义我们对数据的导出操作。创建一个新类

{% highlight ruby %}
    class JsonWithEncodingPipeline(object):
        """
        处理 item 数据，保存为json格式的文件中
        """
        def __init__(self):
            self.file = codecs.open('article.json', 'w', encoding='utf-8')
    
        def process_item(self, item, spider):
            lines = json.dumps(dict(item), ensure_ascii=False) + '\n'   # False，才能够在处理非acsii编码的时候，不会出错，尤其
            #中文
            self.file.write(lines)
            return item     # 必须 return
    
        def spider_close(self, spider):
            """
            把文件关闭
            """
            self.file.close()
{% endhighlight %}

`__init__()` 构造对象的时候，就打开文件，scrapy 会调用 `process_item()` 函数对数据进行处理，在这个函数中，将数据以 json 的格式写入文件中。操作完成之后，将文件关闭。思路很简单。

### scrapy.exporters 的方式
{% highlight ruby %}
    class JsonExporterPipeline(object):

        def __init__(self):
            """
            先打开文件，传递一个文件
            """
            self.file = open('articleexporter.json', 'wb')
            #调用 scrapy 提供的 JsonItemExporter导出json文件
            self.exporter = JsonItemExporter(self.file, encoding="utf-8", ensure_ascii=False)
            self.exporter.start_exporting()
    
        def spider_close(self, spider):
            self.exporter.finish_exporting()
            self.file.close()
    
        def process_item(self, item, spider):
            self.exporter.export_item(item)
            return item
{% endhighlight %}

scrapy.exporters 提供了几种不同格式的文件支持，能够将数据输出到这些不同格式的文件中，查看 JsonItemExporter 源码即可获知

    __all__ = ['BaseItemExporter', 'PprintItemExporter', 'PickleItemExporter',
               'CsvItemExporter', 'XmlItemExporter', 'JsonLinesItemExporter',
               'JsonItemExporter', 'MarshalItemExporter']

这些就是 scrapy 支持的文件。方法名称都差不多，这算是 scrapy 运行 pipeline 的模式，只需要将逻辑处理放在 `process_item()`，scrapy 就会根据规则对数据进行处理。

当然，要想使我们写的数据操作有效，别忘记了，在 settings.py 中进行配置

    ITEM_PIPELINES = {

      'scrapy.pipelines.images.ImagesPipeline': 1,
      'ArticleSpider.pipelines.JsonWithEncodingPipeline': 2,
      'ArticleSpider.pipelines.JsonExporterPipeline':3,
    }

## 扩展五：将数据存储到 MySQL 数据库
前面介绍了将数据以 json 格式导出到文件，那么将数据保存到 MySQL 中，如何操作，相信大家已经差不多了然于胸了。这里也介绍两种方法，一种是通过 MySQLdb 的API来实现的数据库存取操作，这种方法简单，适合用与数据量不大的场合，如果数据量大，数据库操作的速度跟不上数据解析的速度，就会造成数据拥堵。那么使用第二种方法就更好，使用 twisted 框架提供的异步操作方法，不会造成拥堵，速度更快。

既然是入 MySQL 数据库，首先肯定是需要创建数据库表了。表结构如下图所示： <br>
![desc article](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/spider_on_jobble/8.png) <br>
上图中有一个字段的值，我没有讲述怎么取，就是 `front_img_path` 这个值，大家在数据库入库的时候，直接用空置或者空字符串填充即可。这个字段是保存图片在本地保存的路径，这个需要在 ImagesPipe 的 `item_completed(self, results, item, info)` 方法中的 results 参数中获取。

好了，数据库表创建成功之后，下面就来将数据入库了。

### MySQLdb 的方法入库
{% highlight ruby %}
    class MysqlPipeline(object):
        def __init__(self):
            # 连接数据库
            self.conn = MySQLdb.connect('192.168.0.101', 'spider', 'wuzhenyu', 'article_spider', charset="utf8", use_unicode=True)
            self.cursor = self.conn.cursor()
    
        def process_item(self, item, spider):
            insert_sql = """
                insert into article(title, create_date, url, url_object_id, front_img_url, front_img_path, comment_nums, 
                fav_nums, vote_nums, tags, content) VALUES ('%s', '%s', '%s', '%s', '%s', '%s', %d, %d, %d, '%s', '%s')
            """ % (item["title"], item["create_date"], item["url"], item["object_id"],item["front_img_url"],
                   item["front_img_path"], item["comment_nums"], item["fav_nums"], item["vote_nums"], item["tags"],
                   item["content"])
    
            self.cursor.execute(insert_sql)
            self.conn.commit()
    
        def spider_close(self, spider):
            self.cursor.close()
            self.conn.close()
{% endhighlight %}

如果对 API 想了解的更多，就去阅读 python MySQLdb 的相关API文档说明，当然，要想这个生效，首先得在 settings.py 文件中将这个 pipeline 类加入 ITEM_PIPELINE 字典中

    ITEM_PIPELINES = {

      'scrapy.pipelines.images.ImagesPipeline': 1,
      'ArticleSpider.pipelines.JsonWithEncodingPipeline': 2,
      'ArticleSpider.pipelines.JsonExporterPipeline':3,
      'ArticleSpider.pipelines.MysqlPipeline': 4,
    }

### 通过 Twisted 框架提供的异步方法入库
{% highlight ruby %}
    class MysqlTwistedPipeline(object):
        """
        利用 Twisted API 实现异步入库 MySQL 的功能
        Twisted 提供的是一个异步的容器，MySQL 的操作还是使用的MySQLDB 的库
        """
        def __init__(self, dbpool):
            self.dbpool = dbpool
    
        @classmethod
        def from_settings(cls, settings):
            """
            被 spider 调用，将 settings.py 传递进来，读取我们配置的参数
            模仿 images.py 源代码中的 from_settings 函数的写法
            """
            # 字典中的参数，要与 MySQLdb 中的connect 的参数相同
            dbparams = dict(
                host = settings["MYSQL_HOST"],
                db = settings["MYSQL_DBNAME"],
                user = settings["MYSQL_USER"],
                passwd = settings["MYSQL_PASSWORD"],
                charset = "utf8",
                cursorclass = MySQLdb.cursors.DictCursor,
                use_unicode = True
            )
    
            # twisted 中的 adbapi 能够将sql操作转变成异步操作
            dbpool = adbapi.ConnectionPool("MySQLdb", **dbparams)
            return cls(dbpool)
    
        def process_item(self, item, spider):
            """
            使用 twisted 将 mysql 操作编程异步执行
            """
            query = self.dbpool.runInteraction(self.do_insert, item)
            query.addErrback(self.handle_error) # handle exceptions
    
        def handle_error(self, failure):
            """
            处理异步操作的异常
            """
            print(failure)
    
        def do_insert(self, cursor, item):
            """
            执行具体的操作，能够自动 commit
            """
            print(item["create_date"])
            insert_sql = """
                        insert into article(title, create_date, url, url_object_id, front_img_url, front_img_path, comment_nums, 
                        fav_nums, vote_nums, tags, content) VALUES ('%s', '%s', '%s', '%s', '%s', '%s', %d, %d, %d, '%s', '%s');
                    """ % (item["title"], item["create_date"], item["url"], item["object_id"], item["front_img_url"],
                           item["front_img_path"], item["comment_nums"], item["fav_nums"], item["vote_nums"], item["tags"],
                           item["content"])
    
            # self.cursor.execute(insert_sql, (item["title"], item["create_date"], item["url"], item["object_id"],
            #                                 item["front_img_url"], item["front_img_path"], item["comment_nums"],
            #                                 item["fav_nums"], item["vote_nums"], item["tags"], item["content"]))
            print(insert_sql)
            cursor.execute(insert_sql)
{% endhighlight %}
    
博主也是近一个多星期开始学习爬虫的 scrapy 框架，对 Twisted 框架也不怎么熟悉，上面的代码是一个例子，大家可以看下注释，等以后了解更多会补充更多相关知识。

需要提到的是，上面定义的 `from_settings(cls. settings)` 这个类方法， scrapy 会从 settings.py
文件中读取配置进行加载，这里将 MySQL 的一些配置信息放在了 settings.py 文件中，然后使用 `from_settings` 方法直接获取，在 settings.py 中需要添加如下代码：

    # MySQL params
    MYSQL_HOST = ""
    MYSQL_DBNAME = "article_spider"
    MYSQL_USER = "spider"
    MYSQL_PASSWORD = ""
    
> 本篇文章，主要以 scrapy 框架爬取伯乐在线文章为例，简要介绍了 scrapy 爬取数据的一些方法，博主也是最近才开始学习爬虫，有不对的地方还请大家能够指正。
