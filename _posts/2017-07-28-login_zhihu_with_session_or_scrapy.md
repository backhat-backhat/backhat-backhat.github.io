---
layout: post
title:  "使用session或scrapy模拟登录知乎"
date:   2017-07-27 22:18:00
tags: 模拟登录 session scrapy
---
使用session或scrapy模拟登录知乎

# session 模拟登陆
大家知道，登陆知乎需要验证码，那么如何通过 session 来模拟登陆知乎呢。首先通过分析知乎登陆请求， 如下图所示 <br>
![zhihu login](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/login_with_session_and_scrapy/zhihu_login_analysis.png)

上图中标注的部分就是我们需要模拟的请求，也就是说，我们使用 session 登陆的时候，首先需要构造我们的请求头部 header。

在上图中，有一个 `x-xsrftoken` 字段，这个字段是知乎登陆请求中必须有的字段，那么这个字段是如何获取的呢，在我们打开知乎登陆页面的时候，查看网页源代码，在这个里面，我们就能获取到这个信息，如下图所示 <br>
![xsrf token](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/login_with_session_and_scrapy/xsrf_token.png)

这是一个隐藏的标签，在页面上是不会显示的。(这种手法，有点类似于，当关闭浏览器 cookie 功能的时候，服务器传递 `session_id`，也可以采用这种做法，发送一个隐藏的标签，保存 `session_id`)，那么，获取这个 `xsrf token` 就变得非常简单了，首先获取登录页面，然后通过正则表达式获取这个值，代码如下所示：

	def get_xsrf_token():
	    response = session.get("https://www.zhihu.com", headers=header)    # 这个参数必须是 headers
	    match_obj = re.search('.*name="_xsrf" value="(.*?)"', response.text)    # match vs search
	    if match_obj:
	        return match_obj.group(1)
	    else:
	        return ""

大家可能发现，在上面的代码中，我已经使用到了 header，我构造的 header 如下

	agent = "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:54.0) Gecko/20100101 Firefox/54.0"
	header = {
	    "Host": "www.zhihu.com",
	    "Referer": "https://www.zhihu.com/",
	    "Upgrade-Insecure-Requests": "1",
	    "User-Agent": agent
	}

注意，上面的代码的正则表达式部分，使用的是 search，不是 match，match 可能匹配不了正确的结果，不明之处可以查看 python documentation 中的 `match vs search` 的部分

既然 header 和 `xsrf_token` 都已经获取到了，那我们试着使用 session 来模拟登陆一下，在登录之前，我们需要构造 `post_data` 发送用户名和密码进行登录

	def zhihu_login(account, password):
	    """
	    模拟知乎登录
	    """
	    post_data = {}
	    post_url = ""
	    if re.match("^1\d{10}", account):   # 手机号码
	        print("phone login")
	        post_url = "https://www.zhihu.com/login/phone_num"
	        post_data = {
	            "_xsrf": get_xsrf_token(),
	            "phone_num": account,
	            "password": password
	        }
	
	        print(header)
	        print(post_data)
	    else:
	        if "@" in account:
	            print("email login")
	            post_url = "https://www.zhihu.com/login/email"	# 邮箱登录
	            post_data = {
	                "_xsrf": get_xsrf_token(),
	                "email": account,
	                "password": password
	            }
		response_text = session.post(post_url, data=post_data, headers = header)

这段代码运行的时候，登录失败，错误提示是验证码错误，`F12` 浏览器分析，查看登录的参数，如下 <br>
![zhihu login params](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/login_with_session_and_scrapy/login_params.png)

参数包含四个值，分别是 `_xsrf`, `password`, `phone_num`, `captcha_type`，我们在上面的代码中构建的 `post_data` 缺少验证码参数，所以模拟登录失败。

> **知乎登录的时候，如果登录成功，再次登录的时候，可能不需要输入验证码，这是因为在浏览器的 cookies 中保存了相关的数据，F12进入调试模式，将 storage 中的 cookie 删除，再登录，就需要输入验证码了**

同时，还需要注意的一点就是，`captcha_type` 这个参数是 `cn`，验证码是中文，让你点击倒立的汉字，点击正确才能通过验证码验证，如下所示 <br>
![captcha cn](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/login_with_session_and_scrapy/captcha_cn.png)

我们获取验证码的方式比较直接，就是通过将验证码图片下载下来，打开之后，输入验证码，构造 `post_data` 然后发送请求，这样来模拟登录。但是如果使用中文这种点击的方式，就无法采用这种方法了，通过观察参数我们发现，`captcha_type`这个参数为 `cn` 的时候返回的验证码是中文，如果我们将这个值改成 `en` 是不是就是字母了呢，修改后发送请求如下所示(浏览器的调试功能里面，有一个 edit and send 的功能) <br>
![captcha en](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/login_with_session_and_scrapy/captcha_en.png)

获取的验证码就是数字加字母了，这样，我们就能通过先获取验证码，再输入的方式模拟登录了，好了，说了这么多，那么如何获取验证码图片呢？我们在回头来看下验证码的请求 <br>
![get captcha](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/login_with_session_and_scrapy/get_captcha.png)

验证码的请求为`https://www.zhihu.com/captcha.gif?r=1501223240460&type=login&lang=en`，中间 `r=1501223240460` 这个参数是一个随时间变动的参数，可以通过时间 `time.time()` 获取，代码如下

	checkcode_url = "https://www.zhihu.com/captcha.gif?r=" + str(int(time.time()*1000)) + "&type=login&lang=en"
	    resp = session.get(checkcode_url, headers=header).content
	    with open("checkcode.gif", "wb") as f:
	        f.write(resp)
	    f.close()
	
	    try:
	        im = Image.open("checkcode.gif")
	        im.show()
	        im.close()
	    except:
	        pass

> **注意：**这个里面有一个比较大的坑，就是 request 和 session 的问题。只能使用 `session.get`，而不能使用 `requests.get`， 因为一次 session 就是一次会话，在后面继续使用验证码登录时，是在同一个 session 中的，session 会记录很多值，cookie 等。但是，如果使用的是 requests ，前后就不再同一个会话中，那么后面将带有验证码的 `post_data` 发送过去时，会显示验证码无效，登录失败

上面的代码，就是将验证码图片获取之后，保存在 `checkcode.gif` 这个图片中，然后通过 Image 类的 show 方法打开图片，然后后面构造 `post_data` 的时候加上验证码参数即可

	post_data["captcha"] = input("输入验证码： ")

Image 这个类，是在 PIL 这个库中的，如果没有安装，需要先安装

	pip install pillow

然后，在使用的时候，import 进来就可以了

	from PIL import Image

好了，介绍了这么多，知乎模拟登录就完成了。最后，我们可以加上 cookie，在登录之前，加载cookie，如果没有，那么在发送 `post_data` 之后，通过 session 将 cookie 保存起来，在 python2 中，是通过 cookielib 来实现的， python3 中发生了变化，这个方法被移到了 cookiejar 中。完整的代码如下所示

	import requests
	import re
	import time
	
	from PIL import Image
	
	__author__ = ""
	
	# python 中的 cookie lib，能够读取本地的 cookie 文件，生成 cookie，然后赋值给requests 的 cookie
	try:
	    import cookielib    # python 2版本
	except ImportError:
	    import http.cookiejar as cookielib  # python3 版本
	
	session = requests.session()    # 某一次连接，是一个长链接，keep-alive，这样就不需要每一次 request 的时候，都要去建立一次连接
	# 使用 cookielib 实例化 session的cookies，然后才能使用save() 保存cookies，不然在后面就会报错
	
	session.cookies = cookielib.LWPCookieJar(filename="cookies.txt")
	
	try:
	    # 成功加载 load cookie 之后，将不再返回首页
	    session.cookies.load(ignore_discard=True)
	except Exception:
	    print("can not load cookies")
	
	agent = "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:54.0) Gecko/20100101 Firefox/54.0"
	header = {
	    "Host": "www.zhihu.com",
	    "Referer": "https://www.zhihu.com/",
	    "Upgrade-Insecure-Requests": "1",
	    "User-Agent": agent
	}
	
	
	def get_xsrf_token():
	    response = session.get("https://www.zhihu.com", headers=header)    # 这个参数必须是 headers
	    match_obj = re.search('.*name="_xsrf" value="(.*?)"', response.text)    # match vs search
	    if match_obj:
	        return match_obj.group(1)
	    else:
	        return ""
	
	
	def zhihu_login(account, password):
	    """
	    模拟知乎登录
	    """
	    post_data = {}
	    post_url = ""
	    if re.match("^1\d{10}", account):   # 手机号码
	        print("phone login")
	        post_url = "https://www.zhihu.com/login/phone_num"
	        post_data = {
	            "_xsrf": get_xsrf_token(),
	            "phone_num": account,
	            "password": password
	        }
	
	        print(header)
	        print(post_data)
	    else:
	        if "@" in account:
	            print("email login")
	            post_url = "https://www.zhihu.com/login/email"
	            post_data = {
	                "_xsrf": get_xsrf_token(),
	                "email": account,
	                "password": password
	            }
	
	    # get check code image
	    checkcode_url = "https://www.zhihu.com/captcha.gif?r=" + str(int(time.time()*1000)) + "&type=login&lang=en"
	   
	    resp = session.get(checkcode_url, headers=header).content
	    with open("checkcode.gif", "wb") as f:
	        f.write(resp)
	    f.close()
	
	    try:
	        im = Image.open("checkcode.gif")
	        im.show()
	        im.close()
	    except:
	        pass
	
	    post_data["captcha"] = input("checkcode:")
	    response_text = session.post(post_url, data=post_data, headers = header)
	    session.cookies.save()  # 将服务器返回的 cookies 保存到本地，下一次登录的时候，直接从文件中获取

后面进入知乎后，通过 `session.get` 就可以获取登录后的数据。

# scrapy 模拟登录
在 python 中，我们可以通过 `requests.session()` 获取 session，然后模拟登录，但是在 scrapy 中是没有 session，的，那么如何实现模拟登录呢。我使用的是 scrapy 中的 basic 模板。

scrapy 的流程是：

- 入口函数是 `start_requests()`，对 `start_urls` 这个列表中的每一个 url 进行处理，返回一个 Request()，这个函数将 url 给 scrapy 下载器下载，如果没有传递 `callback`，将调用默认的回调函数 `parse` 函数进行解析。

那么，我的思路就是，重载 `start_requests()` 函数，在 `start_requests` 函数中，获取验证码，然后返回 Request 中，调用登录的回调函数，传入 header 和 `post_data`(此时是带有验证码参数的)进行模拟登录，这样就能保证验证码获取后的登录是在一个会话中了。

完整代码如下所示：

    def start_requests(self):
        """
        返回登录页面，在回调函数中，提取 xsrf 的值
        """
        return [scrapy.Request('https://www.zhihu.com/#signin', callback=self.login, headers=self.headers)]

    def login(self, response):
        response_text = response.text
        xsrf = ""

        match_obj = re.search('.*name="_xsrf" value="(.*?)"', response_text)
        if match_obj:
            xsrf = match_obj.group(1)

        if xsrf:
            post_url = "https://www.zhihu.com/login/phone_num"
            post_data = {
                "_xsrf": xsrf,
                "phone_num": "13530210085",
                "password": "wuzhenyu25977758",
                "captcha": ""
            }

        t = str(int(time.time() * 1000))
        checkcode_url = "https://www.zhihu.com/captcha.gif?r={0}&type=login".format(t)

        """
        必须保证验证码的使用，与得到验证码的链接请求是在同一个会话连接中，此处使用 Request 的方法，
        通过 scrapy 将请求下载之后，使用回调函数，在回调函数中进行处理，保证回调函数中的 response 与获取验证码时
        是在同一个会话中
        """
        yield scrapy.Request(checkcode_url, meta={"post_data": post_data}, headers=self.headers,
                             callback=self.login_with_checkcode)

    def login_with_checkcode(self, response):
        post_url = "https://www.zhihu.com/login/phone_num"
        post_data = response.meta.get("post_data", {})
        with open("checkcode.gif", "wb") as f:
            f.write(response.body)
            f.close()

        try:
            im = Image.open("checkcode.gif")
            im.show()
            # im.close()    # no close function
        except:
            pass

        post_data["captcha"] = input("checkcode: ")

        return [scrapy.FormRequest(
            url=post_url,
            formdata=post_data,
            headers=self.headers,
            callback=self.check_login
        )]

    def check_login(self, response):
        """
        检查登录是否成功
        """
        text_json = json.loads(response.text)
        if "msg" in text_json and text_json["msg"] == "登录成功":
            for url in self.start_urls:
                yield scrapy.Request(url, dont_filter=True, headers=self.headers)   # 默认回调函数为 parse

首先，在 `start_requests` 函数中，通过 scrapy 下载登录页面，通过回调函数获取 `xsrf_token` 的值，然后再构造验证码的url，通过 scrapy 下载，注意验证码图片数据此时保存在了 `response.body` 中，通过回调函数 `login_with_checkcode` 构造 `post_data`，然后使用 `scrapy.FormRequest` 模拟登录，在 `check_login` 中对登录结果进行验证，查看登录是否成功。

最后，在 `check_login` 函数中，如果登录成功，再回到之前 `start_requests` 本身应该处理的内容中，将 `start_urls` 中的 url 进行处理，调用默认回调函数 `parse` 对页面进行解析。 `parse` 中是实际的页面解析结果，如何写这部分，大家可以看我的另一篇博客 [爬取伯乐在线所有文章](http://blog.csdn.net/honglicu123/article/details/74906223) 
