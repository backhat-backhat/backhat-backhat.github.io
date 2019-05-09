---
layout: post
title: "redis中 lua 环境的创建和初始化"
date: 2017-04-23 23:30:00
tags: redis lua
---
redis 中 lua 环境解析

redis 中，lua 环境的初始化，是从 `redis.c/initServer()` 函数中，调用 `scriptingInit()` 函数开始的。

关于 `scriptingInit()` 的描述
	
	/* Initialize the scripting environment.
	 * It is possible to call this function to reset the scripting environment
	 * assuming that we call scriptingRelease() before.
	 * See scriptingReset() for more information. */
	 
也就是说，这个函数是初始化lua环境的，当然，如果调用了 `scriptingRelease()` 在调用该函数，可以重置 lua 脚本环境。我们进入到函数中看一下代码。

{% highlight ruby %}
	void scriptingInit(void) {
	    lua_State *lua = lua_open();
	
	    luaLoadLibraries(lua);
	    luaRemoveUnsupportedFunctions(lua);
	
	    /* Initialize a dictionary we use to map SHAs to scripts.
	     * This is useful for replication, as we need to replicate EVALSHA
	     * as EVAL, so we need to remember the associated script. */
	    server.lua_scripts = dictCreate(&shaScriptObjectDictType,NULL);
	
	    /* Register the redis commands table and fields */
	    lua_newtable(lua);
	
	    /* redis.call */
	    lua_pushstring(lua,"call");
	    lua_pushcfunction(lua,luaRedisCallCommand);
	    lua_settable(lua,-3);
	
	    /* redis.pcall */
	    lua_pushstring(lua,"pcall");
	    lua_pushcfunction(lua,luaRedisPCallCommand);
	    lua_settable(lua,-3);
	
	    /* redis.log and log levels. */
	    lua_pushstring(lua,"log");
	    lua_pushcfunction(lua,luaLogCommand);
	    lua_settable(lua,-3);
	
	    lua_pushstring(lua,"LOG_DEBUG");
	    lua_pushnumber(lua,REDIS_DEBUG);
	    lua_settable(lua,-3);
	
	    lua_pushstring(lua,"LOG_VERBOSE");
	    lua_pushnumber(lua,REDIS_VERBOSE);
	    lua_settable(lua,-3);
	
	    lua_pushstring(lua,"LOG_NOTICE");
	    lua_pushnumber(lua,REDIS_NOTICE);
	    lua_settable(lua,-3);
	
	    lua_pushstring(lua,"LOG_WARNING");
	    lua_pushnumber(lua,REDIS_WARNING);
	    lua_settable(lua,-3);
	
	    /* redis.sha1hex */
	    lua_pushstring(lua, "sha1hex");
	    lua_pushcfunction(lua, luaRedisSha1hexCommand);
	    lua_settable(lua, -3);
	
	    /* redis.error_reply and redis.status_reply */
	    lua_pushstring(lua, "error_reply");
	    lua_pushcfunction(lua, luaRedisErrorReplyCommand);
	    lua_settable(lua, -3);
	    lua_pushstring(lua, "status_reply");
	    lua_pushcfunction(lua, luaRedisStatusReplyCommand);
	    lua_settable(lua, -3);
	
	    /* Finally set the table as 'redis' global var. */
	    lua_setglobal(lua,"redis");
	
	    /* Replace math.random and math.randomseed with our implementations. */
	    lua_getglobal(lua,"math");
	
	    lua_pushstring(lua,"random");
	    lua_pushcfunction(lua,redis_math_random);
	    lua_settable(lua,-3);
	
	    lua_pushstring(lua,"randomseed");
	    lua_pushcfunction(lua,redis_math_randomseed);
	    lua_settable(lua,-3);
	
	    lua_setglobal(lua,"math");
	
	    /* Add a helper function that we use to sort the multi bulk output of non
	     * deterministic commands, when containing 'false' elements. */
	    {
	        char *compare_func =    "function __redis__compare_helper(a,b)\n"
	                                "  if a == false then a = '' end\n"
	                                "  if b == false then b = '' end\n"
	                                "  return a<b\n"
	                                "end\n";
	        luaL_loadbuffer(lua,compare_func,strlen(compare_func),"@cmp_func_def");
	        lua_pcall(lua,0,0,0);
	    }
	
	    /* Add a helper function we use for pcall error reporting.
	     * Note that when the error is in the C function we want to report the
	     * information about the caller, that's what makes sense from the point
	     * of view of the user debugging a script. */
	    {
	        char *errh_func =       "local dbg = debug\n"
	                                "function __redis__err__handler(err)\n"
	                                "  local i = dbg.getinfo(2,'nSl')\n"
	                                "  if i and i.what == 'C' then\n"
	                                "    i = dbg.getinfo(3,'nSl')\n"
	                                "  end\n"
	                                "  if i then\n"
	                                "    return i.source .. ':' .. i.currentline .. ': ' .. err\n"
	                                "  else\n"
	                                "    return err\n"
	                                "  end\n"
	                                "end\n";
	        luaL_loadbuffer(lua,errh_func,strlen(errh_func),"@err_handler_def");
	        lua_pcall(lua,0,0,0);
	    }
	
	    /* Create the (non connected) client that we use to execute Redis commands
	     * inside the Lua interpreter.
	     * Note: there is no need to create it again when this function is called
	     * by scriptingReset(). */
	    if (server.lua_client == NULL) {
	        server.lua_client = createClient(-1);
	        server.lua_client->flags |= REDIS_LUA_CLIENT;
	    }
	
	    /* Lua beginners often don't use "local", this is likely to introduce
	     * subtle bugs in their code. To prevent problems we protect accesses
	     * to global variables. */
	    scriptingEnableGlobalsProtection(lua);
	
	    server.lua = lua;
	}
{% endhighlight %}

1、 lua_open() 函数，创建一个lua环境 <br>
2、 `luaLoadLibraries(lua)`，在新创建的lua环境中载入相应的库，同时删除库中不需要的函数，防止从外部引入不安全的代码 `luaRemoveUnsupportedFunctions(lua)`，

	void luaLoadLibraries(lua_State *lua) {
	    luaLoadLib(lua, "", luaopen_base);
	    luaLoadLib(lua, LUA_TABLIBNAME, luaopen_table);
	    luaLoadLib(lua, LUA_STRLIBNAME, luaopen_string);
	    luaLoadLib(lua, LUA_MATHLIBNAME, luaopen_math);
	    luaLoadLib(lua, LUA_DBLIBNAME, luaopen_debug);
	    luaLoadLib(lua, "cjson", luaopen_cjson);
	    luaLoadLib(lua, "struct", luaopen_struct);
	    luaLoadLib(lua, "cmsgpack", luaopen_cmsgpack);
	    luaLoadLib(lua, "bit", luaopen_bit);
	
	#if 0 /* Stuff that we don't load currently, for sandboxing concerns. */
	    luaLoadLib(lua, LUA_LOADLIBNAME, luaopen_package);
	    luaLoadLib(lua, LUA_OSLIBNAME, luaopen_os);
	#endif
	}

	/* Remove a functions that we don't want to expose to the Redis scripting
	 * environment. */
	void luaRemoveUnsupportedFunctions(lua_State *lua) {
	    lua_pushnil(lua);	//将空值入栈，此时栈顶元素为空值
	    lua_setglobal(lua,"loadfile");	//出栈，取出栈顶元素，即空值，并将其作为 loadfile 的值，也就相当于取消了 loadfile 这个函数的作用，置空了
	}
	
3、 `lua_newtable(lua)` 创建一张空表，并入栈 <br>
4、 将 c 中的luaRedisCallCommand 函数压入栈，作为表的函数；将 c 中的luaRedisPCallCommand 函数入栈，作为表的函数；将 c 中的luaLogCommand 函数入栈作为表的函数；将表复制为 redis，即创建 redis 全局 table。下面对其中一个注册函数进行解释

	lua_newtable(lua);

    /* redis.call */
    lua_pushstring(lua,"call");
    lua_pushcfunction(lua,luaRedisCallCommand);
    lua_settable(lua,-3);
	
首先创建一个空表，入栈，此时栈顶元素为 table。然后将字符串 "call" 入栈，再将 c 函数 luaRedisCallCommand 函数入栈

	lua_settable(lua, -3)
	
意思是 t[k]=v，t 为索引-3，栈中，-3 位元素 table，-1 为 luaRedisCallCommand，-2 为字符串call，v 表示栈顶元素，k表示栈顶元素的下一个元素，索引上面这个函数的意思就是

	table[call]=luaRedisCallCommand
	
就是将lua 中的call 函数注册为 c 中的 luaRedisCallCommand 函数，也可以记为

	table.call=luaRedisCallCommand
	
后面代码意思都雷同，都是注册 c 函数到 lua 中。

**注意：** `lua_settable(lua_State *L, int index)` 会将栈顶两个元素弹出。

	lua_setglobal(lua, "redis")
	
将表作为 redis的值，即创建的表为 redis 表，该表中包含以下函数： <br>

- redis.call 函数和 redis.pcall 函数，用户执行 redis 命令
- redis.log 记录日志，日志级别对应为 `redid.LOG_DEBUG`， `redis.LOG_VERBOSE`，`redis.LOG_NOTICE`，`redis.LOG_WARNING`
- redis.sha1hex，计算 SHA1 校验和
- `redis.error_reply` 和 `redis.status_reply` 函数，返回 redis 错误信息


5、 用 c 中自制的随机函数替换 Lua 中原有的随机函数 <br>
6、 创建排序辅助函数， Lua 环境使用这个辅助函数来对一部分 redis 命令的结果进行排序，从而消除这些命令的不确定性。 <br>

{% highlight ruby %}
	function __redis__compare_helper(a,b)
	    if a == false 
		then 
			a = '' 
		end
	    if b == false 
		then 
			b = '' 
		end
	    return a<b
	end
{% endhighlight %}

7、 创建 redis.pcall 函数的错误报告辅助函数，这个函数可以提供更加详细发出错信息，比如能够在 c 函数的出错信息中提供调用者的信息

{% highlight ruby %}
	local dbg = debug;
	function __redis_err_handler(err)
		local i = dbg.getinfo (2, 'sS1')
		if i and i.what = 'C'
		then
			i = dgb.getinfo (3, 'nS1')
		end
		if i
		then
			return i.source .. ':' .. i.currentLine .. ':' .. err
		else
			return err
		end
	end
{% endhighlight %}

8、 对 lua 环境中的全局环境进行保护，防止用户在执行 lua 脚本的过程中，将额外的全局变量添加到 Lua 环境中 <br>
9、 将完成修改的 lua 环境保存到服务器状态的 lua 属性中。

## 创建排序辅助函数 `__redis__compare_helper`
在 redis 中产生不同输出的命令称为“带有不确定性的命令”，比如： <br>

- SINTER
- SUNION
- SDIFF
- SMEMBERS
- HKEYS
- HVALS
- KEYS

比如对 `SMEMBERS` 来说，在两个值相同，但是顺序不同的集合中，使用 SMEMBERS 得到的结果是不同的，输出的值的顺序不同。这样的输出就是不确定性，因为它本身不会排序。而 lua 中创建的这个辅助排序函数，可以用来消除这种不确定性。当 lua 执行完一个带有不确定性的命令时，程序会使用 `__redis__compare_helper` 作为对比函数，自动调用 tables.sort 函数对命令进行一次排序，一次来保证相同的数据集总是产生相同的输出。

那，到底是什么一个代码流程呢，因为 redis.call 和 redis.pcall 在初始化的时候，已经使用 c 中的函数进行了注册，所以当调用 redis.call 或者 redis.pcall 的时候实际唤醒调用的是 c 函数 luaRedisCallCommand 或者 luaRedisPCallCommand 

	int luaRedisCallCommand(lua_State *lua) {
	    return luaRedisGenericCommand(lua,1);
	}
	
	int luaRedisPCallCommand(lua_State *lua) {
	    return luaRedisGenericCommand(lua,0);
	}
	
这两个函数，都是调用的 `luaRedisGenericCommand` 函数，此时，在 `luaGenericCommand` 函数中，在满足条件的情况下，调用 `luaSortArray` 函数。

	if ((cmd->flags & REDIS_CMD_SORT_FOR_SCRIPT) &&
        (reply[0] == '*' && reply[1] != '-')) {
            luaSortArray(lua);
    }
	
满足上面条件的时候，才调用 `luaSortArray` 这个函数，函数定义如下所示

{% highlight ruby %}
	/* Sort the array currently in the stack. We do this to make the output
	 * of commands like KEYS or SMEMBERS something deterministic when called
	 * from Lua (to play well with AOf/replication).
	 *
	 * The array is sorted using table.sort itself, and assuming all the
	 * list elements are strings. */
	void luaSortArray(lua_State *lua) {
	    /* Initial Stack: array */
	    lua_getglobal(lua,"table");
	    lua_pushstring(lua,"sort");
	    lua_gettable(lua,-2);       /* Stack: array, table, table.sort */
	    lua_pushvalue(lua,-3);      /* Stack: array, table, table.sort, array */
	    if (lua_pcall(lua,1,0,0)) {
	        /* Stack: array, table, error */
	
	        /* We are not interested in the error, we assume that the problem is
	         * that there are 'false' elements inside the array, so we try
	         * again with a slower function but able to handle this case, that
	         * is: table.sort(table, __redis__compare_helper) */
	        lua_pop(lua,1);             /* Stack: array, table */
	        lua_pushstring(lua,"sort"); /* Stack: array, table, sort */
	        lua_gettable(lua,-2);       /* Stack: array, table, table.sort */
	        lua_pushvalue(lua,-3);      /* Stack: array, table, table.sort, array */
	        lua_getglobal(lua,"__redis__compare_helper");	//table.sort(list [,compare])
	        /* Stack: array, table, table.sort, array, __redis__compare_helper */
	        lua_call(lua,2,0);
	    }
	    /* Stack: array (sorted), table */
	    lua_pop(lua,1);             /* Stack: array (sorted) */
	}
{% endhighlight %}

将 table.sort 中的 comp 参数作为 `__redis_compare_helper` 辅助排序函数进行排序

# lua 环境协作组件
lua 服务器创建了两个用于与 lua 环境进行写作的组件，分别是负责执行 lua 脚本的 redis 命令的伪客户端和保存 lua 脚本的 `lua_scripts` 字典。

	typedef struct Server {
		...
		/* Scripting */
	    lua_State *lua; /* The Lua interpreter. We use just one for all clients */
	    redisClient *lua_client;   /* The "fake client" to query Redis from Lua */
	    redisClient *lua_caller;   /* The client running EVAL right now, or NULL */
	    dict *lua_scripts;         /* A dictionary of SHA1 -> Lua scripts */
	    mstime_t lua_time_limit;  /* Script timeout in milliseconds */
	    mstime_t lua_time_start;  /* Start time of script, milliseconds time */
	    int lua_write_dirty;  /* True if a write command was called during the
	                             execution of the current script. */
	    int lua_random_dirty; /* True if a random command was called during the
	                             execution of the current script. */
	    int lua_timedout;     /* True if we reached the time limit for script
	                             execution. */
	    int lua_kill;         /* Kill the script if true. */
		...
	};

## 伪客户端
redis 命令执行必须有相应的客户端状态，redis服务器专门为 lua 环境创建了一个伪客户端，server 中的 lua_client 成员就是 lua 的伪客户端，当初始化 lua 环境时，对伪客户端初始化如下(scriptingInit())：

	   /* Create the (non connected) client that we use to execute Redis commands
	     * inside the Lua interpreter.
	     * Note: there is no need to create it again when this function is called
	     * by scriptingReset(). */
	    if (server.lua_client == NULL) {
	        server.lua_client = createClient(-1);
	        server.lua_client->flags |= REDIS_LUA_CLIENT;
	    }
		
Lua 脚本使用 redis.call 或者 redis.pcall 执行 redis 命令的时候，步骤如下： <br>

> - Lua 环境将 redis.call 或者 redis.pcall 想要执行的命令传给伪客户端
- 伪客户端将脚本想要执行的命令传给命令执行器 (call)
- 命令执行器执行命令后，将结果返回给伪客户端
- 伪客户端接收到结果并将结果返回给 lua 环境
- lua 环境接收到结果之后，有将结果返回给 redis.call 或者 redis.pcall 函数
- 接收到结果的 redis.call 或者 redis.pcall 函数，将命令结果作为函数返回值返回给脚本中的调用者。<br>
**以上步骤摘自 《Redis 设计与实现》 黄健宏著，机械工业出版社，20.1.1节 伪客户端**

使用客户端执行lua脚本

	./redis-cli -h ${IP} -p ${PORT} -a ${PASSWORD} --eval <lua script>
	
## lua_scripts 字典
另一个lua环境协作组件是 `lua_scripts` 字典。这个字典的值，是 lua 对应的脚本，键是 lua 脚本的 SHA1 校验和。这个可以在 `scripting.c/luaCreateFunction()` 函数中查看

	  {
	        int retval = dictAdd(server.lua_scripts,
	                             sdsnewlen(funcname+2,40),body);	//dict, key is sha1, value is script
	        redisAssertWithInfo(c,NULL,retval == DICT_OK);
	        incrRefCount(body);
	  }
	  
funcname 就是 lua 脚本的SHA1校验和，body 就是 lua 对应的脚本

# EVAL 命令的实现
**第一步：** <br>
当客户端向服务器发送 EVAL 命令执行一段 lua 脚本的时候，服务器首先在 lua 环境中，为传入的脚本定义一个 lua 函数，函数名由 f_ 前缀加上脚本的 SHA1 校验和，而函数体即为脚本本身。

{% highlight ruby %}
	void evalGenericCommand(redisClient *c, int evalsha) {
	    lua_State *lua = server.lua;
	    char funcname[43];
	    long long numkeys;
	    int delhook = 0, err;
	
	    /* We want the same PRNG sequence at every call so that our PRNG is
	     * not affected by external state. */
	    redisSrand48(0);
	
	    /* We set this flag to zero to remember that so far no random command
	     * was called. This way we can allow the user to call commands like
	     * SRANDMEMBER or RANDOMKEY from Lua scripts as far as no write command
	     * is called (otherwise the replication and AOF would end with non
	     * deterministic sequences).
	     *
	     * Thanks to this flag we'll raise an error every time a write command
	     * is called after a random command was used. */
	    server.lua_random_dirty = 0;
	    server.lua_write_dirty = 0;
	
	    /* Get the number of arguments that are keys */
	    if (getLongLongFromObjectOrReply(c,c->argv[2],&numkeys,NULL) != REDIS_OK)
	        return;
	    if (numkeys > (c->argc - 3)) {
	        addReplyError(c,"Number of keys can't be greater than number of args");
	        return;
	    } else if (numkeys < 0) {
	        addReplyError(c,"Number of keys can't be negative");
	        return;
	    }
	
	    /* We obtain the script SHA1, then check if this function is already
	     * defined into the Lua state */
	    funcname[0] = 'f';
	    funcname[1] = '_';
	    if (!evalsha) {
	        /* Hash the code if this is an EVAL call */
	        sha1hex(funcname+2,c->argv[1]->ptr,sdslen(c->argv[1]->ptr));
	    } else {
	        /* We already have the SHA if it is a EVALSHA */
	        int j;
	        char *sha = c->argv[1]->ptr;
	
	        /* Convert to lowercase. We don't use tolower since the function
	         * managed to always show up in the profiler output consuming
	         * a non trivial amount of time. */
	        for (j = 0; j < 40; j++)
	            funcname[j+2] = (sha[j] >= 'A' && sha[j] <= 'Z') ?
	                sha[j]+('a'-'A') : sha[j];
	        funcname[42] = '\0';
	    }
	
	    /* Push the pcall error handler function on the stack. */
	    lua_getglobal(lua, "__redis__err__handler");
	
	    /* Try to lookup the Lua function */
	    lua_getglobal(lua, funcname);
	    if (lua_isnil(lua,-1)) {
	        lua_pop(lua,1); /* remove the nil from the stack */
	        /* Function not defined... let's define it if we have the
	         * body of the function. If this is an EVALSHA call we can just
	         * return an error. */
	        if (evalsha) {
	            lua_pop(lua,1); /* remove the error handler from the stack. */
	            addReply(c, shared.noscripterr);
	            return;
	        }
	        if (luaCreateFunction(c,lua,funcname,c->argv[1]) == REDIS_ERR) {
	            lua_pop(lua,1); /* remove the error handler from the stack. */
	            /* The error is sent to the client by luaCreateFunction()
	             * itself when it returns REDIS_ERR. */
	            return;
	        }
	        /* Now the following is guaranteed to return non nil */
	        lua_getglobal(lua, funcname);
	        redisAssert(!lua_isnil(lua,-1));
	    }
		...
	}
{% endhighlight %}

首先解析参数，可以查看 EVAL 命令的语法

	EVAL script numkeys key [key ...] arg [arg ...]
	
在 lua 环境创建对应的 lua 函数，保存在变量 funcname 中，

	funcname[0] = 'f';
	funcname[1] = '_';
	
表示函数名以 f_ 作为前缀，使用 shahex1() 函数获取脚本的 SHA1 校验和，保存在 funcname 中，此时的 funcname 将作为 lua 环境中脚本对应的函数名。使用 `luaCreateFunction` 函数在 lua 环境中创建该脚本的 lua 函数，同时将脚本即 lua 环境中对应的函数名加入到 `lua_scripts` 字典中。

{% highlight ruby %}
	/* Define a lua function with the specified function name and body.
	 * The function name musts be a 2 characters long string, since all the
	 * functions we defined in the Lua context are in the form:
	 *
	 *   f_<hex sha1 sum>
	 *
	 * On success REDIS_OK is returned, and nothing is left on the Lua stack.
	 * On error REDIS_ERR is returned and an appropriate error is set in the
	 * client context. */
	int luaCreateFunction(redisClient *c, lua_State *lua, char *funcname, robj *body) {
	    sds funcdef = sdsempty();
	
	    funcdef = sdscat(funcdef,"function ");	//create lua function
	    funcdef = sdscatlen(funcdef,funcname,42);
	    funcdef = sdscatlen(funcdef,"() ",3);
	    funcdef = sdscatlen(funcdef,body->ptr,sdslen(body->ptr));
	    funcdef = sdscatlen(funcdef,"\nend",4);
	
	    if (luaL_loadbuffer(lua,funcdef,sdslen(funcdef),"@user_script")) {
	        addReplyErrorFormat(c,"Error compiling script (new function): %s\n",
	            lua_tostring(lua,-1));
	        lua_pop(lua,1);
	        sdsfree(funcdef);
	        return REDIS_ERR;
	    }
	    sdsfree(funcdef);
	    if (lua_pcall(lua,0,0,0)) {		//execute lua function
	        addReplyErrorFormat(c,"Error running script (new function): %s\n",
	            lua_tostring(lua,-1));
	        lua_pop(lua,1);
	        return REDIS_ERR;
	    }
	
	    /* We also save a SHA1 -> Original script map in a dictionary
	     * so that we can replicate / write in the AOF all the
	     * EVALSHA commands as EVAL using the original script. */
	    {
	        int retval = dictAdd(server.lua_scripts,
	                             sdsnewlen(funcname+2,40),body);	//dict, key is sha1, value is script
	        redisAssertWithInfo(c,NULL,retval == DICT_OK);
	        incrRefCount(body);
	    }
	    return REDIS_OK;
	}
{% endhighlight %}

**第二步：**<br>
执行脚本之前，服务器还需要做一些设置钩子和传入参数的准备工作

a、 将EVAL命令中传入的参数和脚本参数分为保存在 KEYS 和 ARGV 数组中，并作为全局变量保存在 lua 环境中

	/* Populate the argv and keys table accordingly to the arguments that
	     * EVAL received. */
	    luaSetGlobalArray(lua,"KEYS",c->argv+3,numkeys); // KEYS[1]=XX, KEYS[2]=XX
	    luaSetGlobalArray(lua,"ARGV",c->argv+3+numkeys,c->argc-3-numkeys);
		
b、 为 lua 环境装载超时吃力钩子，当脚本运行时间超时时，客户端通过 SCRIPT KILL 命令可以停止脚本，也可以通过 SHUTDOWN 命令停止服务器。

c、 执行脚本 <br>
d、 移除钩子
e、 将执行脚本函数得到的结果保存到客户端状态的输出缓冲去，等待服务器将结果返回给客户端

	if (err) {
	        addReplyErrorFormat(c,"Error running script (call to %s): %s\n",
	            funcname, lua_tostring(lua,-1));
	        lua_pop(lua,2); /* Consume the Lua reply and remove error handler. */
	 } else {
	        /* On success convert the Lua return value into Redis protocol, and
	         * send it to * the client. */
	        luaReplyToRedisReply(c,lua); /* Convert and consume the reply. */
	        lua_pop(lua,1); /* Remove the error handler. */
	 }
	 
# EVALSHA 命令
每一个被 EVAL 命令执行过的脚本，在 lua 环境中都会有一个与脚本对应的 lua 函数，函数的名字由 f_ 前缀和脚本的 SHA1 校验和组成。主要这个函数在 lua 环境中定义了，就会在 `lua_scripts` 中保存，那么使用 EVALSHA 命令，即使不知道脚本本身，也可以直接使用脚本的校验和来调用脚本对应的 lua 环境中的函数，这就是 EVALSHA 实现的原理

# 其他的脚本命令
这里主要讲下命令的作用

__SCRIPT FLUSH__，用于清楚服务器中和 Lua 脚本有关的信息，这个命令会释放并重建 `lua_scripts` 字典，关闭现有的 lua 环境并重新创建一个新的 lua 环境 <br>

__SCRIPT EXISTS__，根据输入的 SHA1 校验和，检查校验和对应的脚本是否存在于服务器中。（注意：该命令允许一次传入多个 SHA1 校验和）

__SCRIPT LOAD__，首先在 lua 环境中为脚本创建对应的 lua 函数，然后将脚本和 SHA1 校验和保存在 `lua_scripts` 字典中

__SCRIPT KILL__，当服务器设置了参数 `lua-time-limit` 时，每次在执行 lua 脚本之前，都会设置一个超时钩子，脚本运行时，一旦钩子发现脚本运行时间已经超时，钩子将定期检查是否有 SCRIPT KILL 或者 SHUTDOWN 命令到达。如果超时脚本未执行任何写操作，客户端可以通过 SCRIPT KILL 命令停止脚本，并向执行脚本的客户端返回一个错误信息。处理完之后，服务器将继续执行。如果脚本已经执行过写操作，那么客户端只能通过 SHUTDOWN 命令停止服务器，防止不合法的数据写入服务器中。

{% highlight ruby %}
	/* ---------------------------------------------------------------------------
	 * SCRIPT command for script environment introspection and control
	 * ------------------------------------------------------------------------- */
	
	void scriptCommand(redisClient *c) {
	    if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"flush")) {
	        scriptingReset();
	        addReply(c,shared.ok);
	        replicationScriptCacheFlush();
	        server.dirty++; /* Propagating this command is a good idea. */
	    } else if (c->argc >= 2 && !strcasecmp(c->argv[1]->ptr,"exists")) {
	        int j;
	
	        addReplyMultiBulkLen(c, c->argc-2);
	        for (j = 2; j < c->argc; j++) {
	            if (dictFind(server.lua_scripts,c->argv[j]->ptr))
	                addReply(c,shared.cone);
	            else
	                addReply(c,shared.czero);
	        }
	    } else if (c->argc == 3 && !strcasecmp(c->argv[1]->ptr,"load")) {
	        char funcname[43];
	        sds sha;
	
	        funcname[0] = 'f';
	        funcname[1] = '_';
	        sha1hex(funcname+2,c->argv[2]->ptr,sdslen(c->argv[2]->ptr));
	        sha = sdsnewlen(funcname+2,40);
	        if (dictFind(server.lua_scripts,sha) == NULL) {
	            if (luaCreateFunction(c,server.lua,funcname,c->argv[2])
	                    == REDIS_ERR) {
	                sdsfree(sha);
	                return;
	            }
	        }
	        addReplyBulkCBuffer(c,funcname+2,40);
	        sdsfree(sha);
	        forceCommandPropagation(c,REDIS_PROPAGATE_REPL|REDIS_PROPAGATE_AOF);
	    } else if (c->argc == 2 && !strcasecmp(c->argv[1]->ptr,"kill")) {
	        if (server.lua_caller == NULL) {
	            addReplySds(c,sdsnew("-NOTBUSY No scripts in execution right now.\r\n"));
	        } else if (server.lua_write_dirty) {
	            addReplySds(c,sdsnew("-UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.\r\n"));
	        } else {
	            server.lua_kill = 1;
	            addReply(c,shared.ok);
	        }
	    } else {
	        addReplyError(c, "Unknown SCRIPT subcommand or wrong # of args.");
	    }
	}
{% endhighlight %}

__参考文献：__ <br>
1. Redis 设计与实现，黄健宏著，机械工业出版社 <br>
2. redis 3.0.7 版本的源代码 <br>
3. [lua manual 5.2](http://www.lua.org/manual/5.2/manual.html)