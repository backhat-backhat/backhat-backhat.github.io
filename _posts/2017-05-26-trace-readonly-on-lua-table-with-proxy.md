---
layout: post
title: "代理的思想-实现lua中table的跟踪与只读"
date: 2017-05-26 23:30:00
tags: lua proxy table 跟踪 只读
---
本文使用 proxy 的思想实现 table 的跟踪与只读

本文跟踪table，是指对一个table 的操作，比如访问和更新进行跟踪。当访问一个 table 或者更新 table 中的某个元素时，lua 首先会在 table 查找是否存在该元素，如果没有，就会查找 table 是否存在 `__index(访问)` 或者 `__newindex(更新)` 原方法。以访问为例，首先在 table 中查找某个字段，如果不存在，解释器会去查找 `__index` 这个原方法，如果仍然没有，返回 **nil**。所以说，`__index` 和 `__newindex` 是在 table 中没有所需访问的 index 时才发挥作用的。

根据上面这种思路，如果我们想跟踪一个 table 的操作行为，那么需要一个空表，每次对这个空表操作的时候，就会使用 `__index` 或者 `__newindex` 这些元方法，在元方法中对原始 table 进行访问和操作，并打印跟踪信息。而之前创建的那个空表，就是代理。

## 对单个 table 的跟踪
先创建一个代理表，为空表，然后创建 metatable

{% highlight ruby %}
	local _t = orignal_table 	-- 对原表的一个私有访问，原表在其他地方创建的
	local proxy = {}
	local mt = {
		__index = function (proxy, k)
			print("access to element " .. tostring(k))
			return _t[k]	-- 这里访问的其实就是原表中的元素了
		end,
		__newindex = function (proxy, k, v)
			print("update of element " .. tostring(k) .. " to " .. tostring(v))
			_t[k] = v		-- 更新原表中的元素
		end
	}
	
	setmetatable(proxy, mt)
	
	-- 操作，观察结果
	proxy[2] = "lua"
	print(proxy[2])
{% endhighlight %}
	
结果为：

	update of element 2 to lua
	access to element 2
	lua

## 对多个table的跟踪
对多个table的跟踪，思路与上面相同，只需要按照某种形式将每个代理与原表 table 关联起来，并且所有的代理都共享一个公共的元表 metatable，比如将原表 table 保存在代理 table 的一个特殊字段中

{% highlight ruby %}
	proxy[index] = t 	-- t 就是原表 table
	
代码如下所示：

	local index = {}	--	创建私有索引，即原表在代理表中特殊字段
	
	local mt = {
		__index = function (t, k)
			print("access to element " .. tostring(k))
			return t[index][k]
		end,
		__newindex = function (t, k, v)
			print("update of element " .. tostring(k) .. " to " .. tostring(v))
			t[index][k] = v
		end
	}
	
	function track (t)
		local proxy = {}
		proxy[index] = t
		setmetatable(proxy, mt)
		return proxy
	end
	
	-- ori_table = {} 在其他地方创建的原表，对他进行跟踪
	local _o = track(ori_table)
	
	_o[2] = "lua"
	print(_o[2])
{% endhighlight %}

观察输出结果，与上面的相同

## 只读 table
只读 table，通过上面的代理的思想也很容易实现，对元方法 `__index` ，访问自身，对更新操作的元方法 `__newindex`，引发一个错误，**不允许更新**，如下所示：

{% highlight ruby %}
	function readOnly (t)
		local proxy = {}
	
		local mt = {
			__index = t,
			__newindex = function (t, k, v)
				error("attempt to update a read-only table", 2)
			end
		}
		setmetatable(proxy, mt)
		return proxy
	end
{% endhighlight %}

因为 proxy 代理 table 是一个空表，当访问时，解释器通过元方法 `__index` 访问，这个元方法此处就是一个 table ，就是原表 table ，直接返回原表 table 中的字段。如果更新字段，解释器将使用元方法 `__newindex` 进行更新，触发一个错误，不允许更新，并说明这是只读的 table。

**参考文献：** <br>
1. 《Programming In Lua》