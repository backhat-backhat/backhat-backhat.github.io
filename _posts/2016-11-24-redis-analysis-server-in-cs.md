---
layout: post
title: "redis源码分析 - cs结构分析之服务器"
date: 2016-11-24 23:30:00
tags: redis code cs server socket
---
对 redis 中，cs结构的服务器的分析。

# 服务器与客户端是如何交互的
redis客户端向服务器发送命令请求，服务器接收到客户端发送的命令请求之后，读取解析命令，并执行命令，同时将命令执行结果返回给客户端。

客户端与服务器交互的代码流程如下图所示： <br>
![代码流程](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/redis_analysis/redis_clientServer_flow.png)

Redis 服务器负责与多个客户端建立网络连接，处理客户端发送的命令请求，在数据库中保存客户端执行的命令产生的数据，并通过资源管理器来维护服务器自身的运转。

redis服务器是一个事件驱动程序，主要为文件事件(File Event)和时间事件(Time Event)。当启动服务器时，服务器在初始化的过程中，会创建时间事件和文件事件，并将对应的事件与事件处理函数绑定，当客户端请求服务器连接或者发送命令请求时，服务器端会触发相应的事件，通过事件处理函数处理完毕后，由服务器通过应答处理事件返回给客户端。时间事件有定时事件和周期性事件两种。

# 服务器中的事件驱动
redis服务器主要处理两种事件： <br>

* 文件事件：这是服务器对套接字操作的抽象，服务器与客户端的通信会产生相应的文件事件，服务器通过监听并处理这些事件来完成一系列的网络通信操作。
* 时间事件：redis中的一些操作需要在指定的时间点执行，时间事件就是服务器对这些定点操作的抽象。

## 文件事件
redis 基于 Reactor 模式开发了自己的网络事件处理器，称为文件事件处理器。
### 文件事件处理器的构成
redis文件事件处理器分为四个组成部分，套接字、IO多路复用程序、文件事件分派器和时间处理函数。

文件事件是对套接字的抽象，当一个套接字准备好执行连接应答(accept)、写入(write)、读取(read)、关闭(close)操作时，就会产生一个文件事件，服务器通过IO多路复用同时监听多个套接字，当这些监听的套接字产生文件事件时，通过轮询的方式，文件事件分派器会对这些文件事件启动相应的事件处理函数。在aeProcessEvents函数中，通过循环的方式，对每一个文件事件进行处理。IO多路复用总是将所有产生事件的套接字都放在一个队列里面，然后按照顺序每次一个套接字的方式向文件事件分派器传送套接字，当上一个套接字产生的事件处理完毕之后，才会处理下一个套接字的文件事件。

### IO多路复用程序
redis中的IO多路复用程序的功能是通过包装 select、epoll、evpoll 和 kqueue 这些IO多路复用库函数来实现的，在源码中对应的文件名为 `ae_select.c`、`ae_epoll.c`、`ae_evpoll.c`、`ae_kqueue.c`。redis在封装这些库函数时，都使用了相同的API，类似于C++的多态实现，这样，IO多路复用程序的底层实现就能够互换。代码如下所示

	/* Include the best multiplexing layer supported by this system.
	 * The following should be ordered by performances, descending. */
	#ifdef HAVE_EVPORT
	#include "ae_evport.c"
	#else
	    #ifdef HAVE_EPOLL
	    #include "ae_epoll.c"
	    #else
	        #ifdef HAVE_KQUEUE
	        #include "ae_kqueue.c"
	        #else
	        #include "ae_select.c"
	        #endif
	    #endif
	#endif
	
通过宏定义规则，在编译时自动选择系统中性能最高的IO多路复用库函数作为redis底层IO多路复用程序的实现，这种方法很巧妙。

### 文件事件处理器的实现
**问题：redis文件事件处理器是由套接字、IO多路复用程序、文件事件分派器和事件处理函数组成，那么套接字能够产生哪些事件，事件处理函数又有哪些操作呢？**

#### 事件的类型
在redis中，文件事件创建函数为 `aeCreateFileEvent`，其函数如下

{% highlight ruby %}
	/* 
	fd : socket file descriptor
	mask : type of event, READABLE or WRITABLE
	proc : handler of file event
	 */
	int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
	        aeFileProc *proc, void *clientData)
	{
	    if (fd >= eventLoop->setsize) {
	        errno = ERANGE;
	        return AE_ERR;
	    }
	    aeFileEvent *fe = &eventLoop->events[fd];
	
		/* add file event, attach fd to event 
		   if type is READABLE, add fd to fd_set rfds, else if type is WRITABLE, add to wfds
		 */
	    if (aeApiAddEvent(eventLoop, fd, mask) == -1)	
	        return AE_ERR;
	    fe->mask |= mask;
	    if (mask & AE_READABLE) fe->rfileProc = proc;	//now fd attach to file event
	    if (mask & AE_WRITABLE) fe->wfileProc = proc;
	    fe->clientData = clientData;
	    if (fd > eventLoop->maxfd)
	        eventLoop->maxfd = fd;
	    return AE_OK;
	}
{% endhighlight %}

mask 为事件的类型，为 `AE_WRITABLE` 和 `AE_READABLE` 两种，分别为可写和可读两种类型。proc 为文件事件处理函数，fd 为套接字的文件描述符，而 clientData 则是客户端在服务器端的状态信息，这个后面会重点讲述。

也就是说，文件事件的类型分为可读和可写两种类型，当然，同一个套接字是允许同时产生这两种类型的事件的。

**问题：那么，什么时候，套接字产生的文件是可读的，什么时候是可写的呢？**

1. 当客户端对服务器发起连接请求（即客户端对服务器监听的套接字执行connect操作），或者客户端对套接字执行 write 或 close 操作时，套接字变的可读，此时产生可读事件 (AE_READABLE)。
2. 当客户端对套接字执行 read 操作时，套接字变得可写，此时产生可写事件 (AE_WRITABLE)。

通过查看 `ae_select.c/aeApiPoll` 函数理解服务器是如何监听套接字的文件事件的。

{% highlight ruby %}
	static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
	    aeApiState *state = eventLoop->apidata;
	    int retval, j, numevents = 0;
	
	    memcpy(&state->_rfds,&state->rfds,sizeof(fd_set));
	    memcpy(&state->_wfds,&state->wfds,sizeof(fd_set));
	
		/* allow a program to monitor multiple file descriptors, waiting until 
		   one or more of the file descriptors become "ready" */
	    retval = select(eventLoop->maxfd+1,
	                &state->_rfds,&state->_wfds,NULL,tvp);
	    if (retval > 0) {
	        for (j = 0; j <= eventLoop->maxfd; j++) {
	            int mask = 0;
	            aeFileEvent *fe = &eventLoop->events[j];
	
	            if (fe->mask == AE_NONE) continue;
	            if (fe->mask & AE_READABLE && FD_ISSET(j,&state->_rfds))
	                mask |= AE_READABLE;
	            if (fe->mask & AE_WRITABLE && FD_ISSET(j,&state->_wfds))
	                mask |= AE_WRITABLE;
	            eventLoop->fired[numevents].fd = j;
	            eventLoop->fired[numevents].mask = mask;
	            numevents++;
	        }
	    }
	    return numevents;
	}
{% endhighlight %}

返回值 numevents，为产生的文件事件的个数。通过多路复用IO库函数 select，监听多个套接字，当套接字符合上述要求时，会变得可读或者可写，可读的套接字保存在套接字集合 `state->_rfds` 中，可写的保存在 `state->_wfds` 中，异常情况的套接字集合设置为 NULL，这里不关心。然后根据套接字的可读或者可写状态，预设文件事件，将他们的文件描述符fd 和 事件类型 mask 保存在 fired 数组中，这个数组中保存的都是产生事件的套接字，然后通过扫描 fired 数组，对产生的文件事件一个一个的进行处理。

> 如果对 select 函数不了解，可查看 **select函数详解及实例解析**[http://blog.csdn.net/leo115/article/details/8097143]

在 `aeProcessEvents` 函数中，通过调用上述的 `aeApiPoll` 函数，等待和分配文件事件，然后调用对应的事件处理函数进行处理。

{% highlight ruby %}
	/* Process every pending time event, then every pending file event
	 * (that may be registered by time event callbacks just processed).
	 * Without special flags the function sleeps until some file event
	 * fires, or when the next time event occurs (if any).
	 *
	 * If flags is 0, the function does nothing and returns.
	 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
	 * if flags has AE_FILE_EVENTS set, file events are processed.
	 * if flags has AE_TIME_EVENTS set, time events are processed.
	 * if flags has AE_DONT_WAIT set the function returns ASAP until all
	 * the events that's possible to process without to wait are processed.
	 *
	 * The function returns the number of events processed. */
	 /* the event dispatcher, make a choice to select an event handler */
	int aeProcessEvents(aeEventLoop *eventLoop, int flags)
	{
	    int processed = 0, numevents;
	
	    /* Nothing to do? return ASAP */
	    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;
	
	    /* Note that we want call select() even if there are no
	     * file events to process as long as we want to process time
	     * events, in order to sleep until the next time event is ready
	     * to fire. */
	    if (eventLoop->maxfd != -1 ||
	        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
	        int j;
	        aeTimeEvent *shortest = NULL;
	        struct timeval tv, *tvp;
	
	        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
	            shortest = aeSearchNearestTimer(eventLoop);
	        if (shortest) {
	            long now_sec, now_ms;
	
	            /* Calculate the time missing for the nearest
	             * timer to fire. */
	            aeGetTime(&now_sec, &now_ms);
	            tvp = &tv;
	            tvp->tv_sec = shortest->when_sec - now_sec;
	            if (shortest->when_ms < now_ms) {
	                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
	                tvp->tv_sec --;
	            } else {
	                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
	            }
	            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
	            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
	        } else {
	            /* If we have to check for events but need to return
	             * ASAP because of AE_DONT_WAIT we need to set the timeout
	             * to zero */
	            if (flags & AE_DONT_WAIT) {
	                tv.tv_sec = tv.tv_usec = 0;
	                tvp = &tv;
	            } else {
	                /* Otherwise we can block */
	                tvp = NULL; /* wait forever */
	            }
	        }
	
	        numevents = aeApiPoll(eventLoop, tvp);	//how many events fired
	        for (j = 0; j < numevents; j++) {
	            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
	            int mask = eventLoop->fired[j].mask;
	            int fd = eventLoop->fired[j].fd;
	            int rfired = 0;
	
		    /* note the fe->mask & mask & ... code: maybe an already processed
	             * event removed an element that fired and we still didn't
	             * processed, so we check if the event is still valid. */
	            if (fe->mask & mask & AE_READABLE) {
	                rfired = 1;
	                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
	            }
	            if (fe->mask & mask & AE_WRITABLE) {
	                if (!rfired || fe->wfileProc != fe->rfileProc)
	                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
	            }
	            processed++;
	        }
	    }
	    /* Check time events */
	    if (flags & AE_TIME_EVENTS)
	        processed += processTimeEvents(eventLoop);
	
	    return processed; /* return the number of processed file/time events */
	}
{% endhighlight %}

`aeProcessEvents` 函数，先处理文件事件，如果此时时间事件触发，在处理时间事件。`aeProcessEvents` 就是时间分派器，将产生的文件事件分派给对应的事件处理函数进行处理。


### 事件处理函数
现在再来回顾一下，redis文件事件处理器的构成，套接字、IO多路复用程序、事件分派器和事件处理函数。如下图所示： <br>
![components of file events](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/redis_analysis/file_events_components.png)

redis 服务器中，事件处理函数，主要由上图中列出的三种，连接应答处理器(acceptTcpHandler)、命令请求处理器(readQueryFromClient)和命令回复处理器(sendReplyToClient)。这里所说的都是文件事件处理函数。

__连接应答处理器 acceptTcpHandler__ <br>
用于对服务器监听的套接字请求连接的客户端进行应答(即客户端执行connect)，具体实现为 `accept()` 函数的封装。

	void initServer (void)
	{
		...
    	/* Create an event handler for accepting new connections in TCP and Unix
    	 * domain sockets. */
    	for (j = 0; j < server.ipfd_count; j++) {
       		if (aeCreateFileEvent(server.el, server.ipfd[j], 	AE_READABLE,
            	acceptTcpHandler,NULL) == AE_ERR) {
                	redisPanic(
                    	"Unrecoverable error creating server.ipfd file event.");
            	}
    	}
		...
	} 

redis 在初始化时，会创建文件事件，将连接应答处理器与服务器监听的套接字的 `AE_READABLE` 类型的事件关联起来(或者说是绑定)，当客户端连接服务器(connect)时，该被服务器监听的套接字会会变成 `AE_READABLE` ，IO多路复用程序将该套接字保存在可读的套接字集合中，引发连接应答处理器执行相应的操作。

{% highlight ruby %}
	void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
	    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
	    char cip[REDIS_IP_STR_LEN];
	    REDIS_NOTUSED(el);
	    REDIS_NOTUSED(mask);
	    REDIS_NOTUSED(privdata);
	
	    while(max--) {
	        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
	        if (cfd == ANET_ERR) {
	            if (errno != EWOULDBLOCK)
	                redisLog(REDIS_WARNING,
	                    "Accepting client connection: %s", server.neterr);
	            return;
	        }
	        redisLog(REDIS_VERBOSE,"Accepted %s:%d", cip, cport);
	        acceptCommonHandler(cfd,0);
	    }
	}
{% endhighlight %}

__命令请求处理器 readQueryFromClient__ <br>
负责读取客户端发送的命令请求内容，底层实现为 `read` 函数的封装。当客户端通过连接应答处理器成功连接服务器之后，服务器会将命令请求处理器与套接字的 `AE_READABLE` 关联起来，当客户端向服务器发送命令请求的时候，套接字就产生了 `AE_READABLE` 类型的文件事件，触发命令请求处理器，由该处理器对套接字执行相应的操作。

在服务器端，会有一个 `redisClient` 结构，用于保存客户端的状态信息。

{% highlight ruby %}
	static void acceptCommonHandler(int fd, int flags) {
	    redisClient *c;
	    if ((c = createClient(fd)) == NULL) {
	        redisLog(REDIS_WARNING,
	            "Error registering fd event for the new client: %s (fd=%d)",
	            strerror(errno),fd);
	        close(fd); /* May be already closed, just ignore errors */
	        return;
	    }
	    /* If maxclient directive is set and this is one client more... close the
	     * connection. Note that we create the client instead to check before
	     * for this condition, since now the socket is already set in non-blocking
	     * mode and we can send an error for free using the Kernel I/O */
	    if (listLength(server.clients) > server.maxclients) {
	        char *err = "-ERR max number of clients reached\r\n";
	
	        /* That's a best effort error message, don't check write errors */
	        if (write(c->fd,err,strlen(err)) == -1) {
	            /* Nothing to do, Just to avoid the warning... */
	        }
	        server.stat_rejected_conn++;
	        freeClient(c);
	        return;
	    }
	    server.stat_numconnections++;
	    c->flags |= flags;
	}
{% endhighlight %}

连接应答处理器连接成功时，会处理上述函数，函数的主要功能，是当客户端成功连接服务器时，就创建一个新的客户端类型的对象(redisClient)用于保存客户端的信息，同时，将该客户端加入到服务器的客户端链表中。

{% highlight ruby %}
	redisClient *createClient(int fd) {
	    redisClient *c = zmalloc(sizeof(redisClient));
	
	    /* passing -1 as fd it is possible to create a non connected client.
	     * This is useful since all the Redis commands needs to be executed
	     * in the context of a client. When commands are executed in other
	     * contexts (for instance a Lua script) we need a non connected client. */
	    if (fd != -1) {
	        anetNonBlock(NULL,fd);
	        anetEnableTcpNoDelay(NULL,fd);
	        if (server.tcpkeepalive)
	            anetKeepAlive(NULL,fd,server.tcpkeepalive);
	        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
	            readQueryFromClient, c) == AE_ERR)
	        {
	            close(fd);
	            zfree(c);
	            return NULL;
	        }
		}
		/* 对客户端其他信息的初始化 */
	    ...
	}
{% endhighlight %}

而在创建客户端时，就会创建文件事件，将套接字的 `AE_READABLE` 与命令请求处理器关联。如上述函数所示。

{% highlight ruby %}
	void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
	    redisClient *c = (redisClient*) privdata;
	    int nread, readlen;
	    size_t qblen;
	    REDIS_NOTUSED(el);
	    REDIS_NOTUSED(mask);
	
	    server.current_client = c;
	    readlen = REDIS_IOBUF_LEN;
	    /* If this is a multi bulk request, and we are processing a bulk reply
	     * that is large enough, try to maximize the probability that the query
	     * buffer contains exactly the SDS string representing the object, even
	     * at the risk of requiring more read(2) calls. This way the function
	     * processMultiBulkBuffer() can avoid copying buffers to create the
	     * Redis Object representing the argument. */
	    if (c->reqtype == REDIS_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
	        && c->bulklen >= REDIS_MBULK_BIG_ARG)
	    {
	        int remaining = (unsigned)(c->bulklen+2)-sdslen(c->querybuf);
	
	        if (remaining < readlen) readlen = remaining;
	    }
	
	    qblen = sdslen(c->querybuf);
	    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
	    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
	    nread = read(fd, c->querybuf+qblen, readlen);
	    if (nread == -1) {
	        if (errno == EAGAIN) {
	            nread = 0;
	        } else {
	            redisLog(REDIS_VERBOSE, "Reading from client: %s",strerror(errno));
	            freeClient(c);
	            return;
	        }
	    } else if (nread == 0) {
	        redisLog(REDIS_VERBOSE, "Client closed connection");
	        freeClient(c);
	        return;
	    }
	    if (nread) {
	        sdsIncrLen(c->querybuf,nread);
	        c->lastinteraction = server.unixtime;
	        if (c->flags & REDIS_MASTER) c->reploff += nread;
	        server.stat_net_input_bytes += nread;
	    } else {
	        server.current_client = NULL;
	        return;
	    }
		/* 如果缓冲区超出最大范围，关闭该客户端 */
	    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {	//max query buf is 1GB
	        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();
	
	        bytes = sdscatrepr(bytes,c->querybuf,64);
	        redisLog(REDIS_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
	        sdsfree(ci);
	        sdsfree(bytes);
	        freeClient(c);
	        return;
	    }
	    processInputBuffer(c);	//解析c->querybuf 中的参数，以redisStringObject的方式放入 c->argv 数组中
	    server.current_client = NULL;
	}
{% endhighlight %}

`readQueryFromClient` 函数，读取客户端发送的命令请求，存放在 `c->querybuf` 中，这是客户端缓冲区，最大限制为 `REDIS_MAX_QUERYBUF_LEN`，这个宏定义为 `redis.h`

	#define REDIS_MAX_QUERYBUF_LEN  (1024*1024*1024) /* 1GB max query buffer. */
也就是说客户端缓冲区最大为 1GB，如果超过这个大小，服务器将会关闭这个客户端。`processInputBuffer` 函数是对客户端缓冲区中的命令请求进行解析。

__命令回复处理器__ <br>
负责将服务器执行命令后得到的结果通过套接字返回给客户端。底层实现为 `write` 函数的封装。当服务器执行命令结果需要返回给客户端的时候，服务器就会创建文件事件，将命令回复处理器和套接字的 `AE_WRITABLE` 类型的时间关联起来。当客户端需要接受服务器传回的结果时，就会产生 `AE_WRITABLE` 类型的文件事件，引发命令回复处理器执行，对套接字进行操作。

{% highlight ruby %}
	/* This function is called every time we are going to transmit new data
	 * to the client. The behavior is the following:
	 *
	 * If the client should receive new data (normal clients will) the function
	 * returns REDIS_OK, and make sure to install the write handler in our event
	 * loop so that when the socket is writable new data gets written.
	 *
	 * If the client should not receive new data, because it is a fake client
	 * (used to load AOF in memory), a master or because the setup of the write
	 * handler failed, the function returns REDIS_ERR.
	 *
	 * The function may return REDIS_OK without actually installing the write
	 * event handler in the following cases:
	 *
	 * 1) The event handler should already be installed since the output buffer
	 *    already contained something.
	 * 2) The client is a slave but not yet online, so we want to just accumulate
	 *    writes in the buffer but not actually sending them yet.
	 *
	 * Typically gets called every time a reply is built, before adding more
	 * data to the clients output buffers. If the function returns REDIS_ERR no
	 * data should be appended to the output buffers. */
	int prepareClientToWrite(redisClient *c) {
	    /* If it's the Lua client we always return ok without installing any
	     * handler since there is no socket at all. */
	    if (c->flags & REDIS_LUA_CLIENT) return REDIS_OK;
	
	    /* Masters don't receive replies, unless REDIS_MASTER_FORCE_REPLY flag
	     * is set. */
	    if ((c->flags & REDIS_MASTER) &&
	        !(c->flags & REDIS_MASTER_FORCE_REPLY)) return REDIS_ERR;
	
	    if (c->fd <= 0) return REDIS_ERR; /* Fake client for AOF loading. */
	
	    /* Only install the handler if not already installed and, in case of
	     * slaves, if the client can actually receive writes. */
	    if (c->bufpos == 0 && listLength(c->reply) == 0 &&
	        (c->replstate == REDIS_REPL_NONE ||
	         (c->replstate == REDIS_REPL_ONLINE && !c->repl_put_online_on_ack)))
	    {
	        /* Try to install the write handler. */
	        if (aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
	                sendReplyToClient, c) == AE_ERR)
	        {
	            freeClientAsync(c);
	            return REDIS_ERR;
	        }
	    }
	
	    /* Authorize the caller to queue in the output buffer of this client. */
	    return REDIS_OK;
	}
{% endhighlight %}

`sendReplyToClient` 函数就是将命令结果返回到客户端

{% highlight ruby %}
	void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask) {
	    redisClient *c = privdata;
	    int nwritten = 0, totwritten = 0, objlen;
	    size_t objmem;
	    robj *o;
	    REDIS_NOTUSED(el);
	    REDIS_NOTUSED(mask);
	
	    while(c->bufpos > 0 || listLength(c->reply)) {
	        if (c->bufpos > 0) {
	            nwritten = write(fd,c->buf+c->sentlen,c->bufpos-c->sentlen);
	            if (nwritten <= 0) break;
	            c->sentlen += nwritten;
	            totwritten += nwritten;
	
	            /* If the buffer was sent, set bufpos to zero to continue with
	             * the remainder of the reply. */
	            if (c->sentlen == c->bufpos) {
	                c->bufpos = 0;
	                c->sentlen = 0;
	            }
	        } else {
	            o = listNodeValue(listFirst(c->reply));
	            objlen = sdslen(o->ptr);
	            objmem = getStringObjectSdsUsedMemory(o);
	
	            if (objlen == 0) {
	                listDelNode(c->reply,listFirst(c->reply));
	                c->reply_bytes -= objmem;
	                continue;
	            }
	
	            nwritten = write(fd, ((char*)o->ptr)+c->sentlen,objlen-c->sentlen);
	            if (nwritten <= 0) break;
	            c->sentlen += nwritten;
	            totwritten += nwritten;
	
	            /* If we fully sent the object on head go to the next one */
	            if (c->sentlen == objlen) {
	                listDelNode(c->reply,listFirst(c->reply));
	                c->sentlen = 0;
	                c->reply_bytes -= objmem;
	            }
	        }
	        /* Note that we avoid to send more than REDIS_MAX_WRITE_PER_EVENT
	         * bytes, in a single threaded server it's a good idea to serve
	         * other clients as well, even if a very large request comes from
	         * super fast link that is always able to accept data (in real world
	         * scenario think about 'KEYS *' against the loopback interface).
	         *
	         * However if we are over the maxmemory limit we ignore that and
	         * just deliver as much data as it is possible to deliver. */
	        server.stat_net_output_bytes += totwritten;
	        if (totwritten > REDIS_MAX_WRITE_PER_EVENT &&
	            (server.maxmemory == 0 ||
	             zmalloc_used_memory() < server.maxmemory)) break;
	    }
	    if (nwritten == -1) {
	        if (errno == EAGAIN) {
	            nwritten = 0;
	        } else {
	            redisLog(REDIS_VERBOSE,
	                "Error writing to client: %s", strerror(errno));
	            freeClient(c);
	            return;
	        }
	    }
	    if (totwritten > 0) {
	        /* For clients representing masters we don't count sending data
	         * as an interaction, since we always send REPLCONF ACK commands
	         * that take some time to just fill the socket output buffer.
	         * We just rely on data / pings received for timeout detection. */
	        if (!(c->flags & REDIS_MASTER)) c->lastinteraction = server.unixtime;
	    }
	    if (c->bufpos == 0 && listLength(c->reply) == 0) {
	        c->sentlen = 0;
	        aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);
	
	        /* Close connection after entire reply has been sent. */
	        if (c->flags & REDIS_CLOSE_AFTER_REPLY) freeClient(c);
	    }
	}
{% endhighlight %}

## 时间事件
	/* Time event structure */
	typedef struct aeTimeEvent {
	    long long id; /* time event identifier. , 值是递增的 */
	    long when_sec; /* seconds , 时间事件达到时间，秒精度 */
	    long when_ms; /* milliseconds, 毫秒精度 */
	    aeTimeProc *timeProc;	/* 时间事件处理函数 */
	    aeEventFinalizerProc *finalizerProc;
	    void *clientData;
	    struct aeTimeEvent *next;	/* 时间事件，以链表的形式连接 */
	} aeTimeEvent;

时间事件分为两种，一个是定时事件，一个是周期性事件。 <br>
__定时事件__：让一段程序在指定一段时间之后执行 <br>
__周期性事件__：让一段程序每隔指定时间执行一次。

### 创建时间事件
	long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
	        aeTimeProc *proc, void *clientData,
	        aeEventFinalizerProc *finalizerProc)
	{
	    long long id = eventLoop->timeEventNextId++;
	    aeTimeEvent *te;
	
	    te = zmalloc(sizeof(*te));
	    if (te == NULL) return AE_ERR;
	    te->id = id;
	    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
	    te->timeProc = proc;
	    te->finalizerProc = finalizerProc;
	    te->clientData = clientData;
	    te->next = eventLoop->timeEventHead;	//strat from zero
	    eventLoop->timeEventHead = te;
	    return id;
	}

`milliseconds`：是多久之后执行时间事件的参数 <br>
`id`：是时间事件的唯一 id 标识，从 0 开始计数 <br>

`aeAddMillisecondsToNow` 函数用于更新时间事件的 when_sec 和 when_ms 变量，即用当前时间加上 milliseconds 转换的时间，表示 milliseconds 时间之后将会执行该时间事件。

### 删除时间事件
	int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id)
	{
	    aeTimeEvent *te, *prev = NULL;
	
	    te = eventLoop->timeEventHead;
	    while(te) {
	        if (te->id == id) {
	            if (prev == NULL)
	                eventLoop->timeEventHead = te->next;
	            else
	                prev->next = te->next;
	            if (te->finalizerProc)
	                te->finalizerProc(eventLoop, te->clientData);
	            zfree(te);
	            return AE_OK;
	        }
	        prev = te;
	        te = te->next;
	    }
	    return AE_ERR; /* NO event with the specified ID found */
	}

在 redis 中，多个时间事件是通过单链表连接起来的，链表头结点为 `eventLoop->timeEventHead`，删除时间事件时，先通过 id 找到时间事件，然后在单链表中删除该节点。

### 查找当前时间最近的时间事件
	/* Search the first timer to fire.
	 * This operation is useful to know how many time the select can be
	 * put in sleep without to delay any event.
	 * If there are no timers NULL is returned.
	 *
	 * Note that's O(N) since time events are unsorted.
	 * Possible optimizations (not needed by Redis so far, but...):
	 * 1) Insert the event in order, so that the nearest is just the head.
	 *    Much better but still insertion or deletion of timers is O(N).
	 * 2) Use a skiplist to have this operation as O(1) and insertion as O(log(N)).
	 */
	static aeTimeEvent *aeSearchNearestTimer(aeEventLoop *eventLoop)
	{
	    aeTimeEvent *te = eventLoop->timeEventHead;
	    aeTimeEvent *nearest = NULL;
	
	    while(te) {
	        if (!nearest || te->when_sec < nearest->when_sec ||
	                (te->when_sec == nearest->when_sec &&
	                 te->when_ms < nearest->when_ms))
	            nearest = te;
	        te = te->next;
	    }
	    return nearest;
	}

当创建一个时间事件时，将该事件加入到时间事件单链表中，查找链表中离当前时间最近的事件，需要扫描整个链表，类似于一次冒泡排序。

### 时间事件的调度
{% highlight ruby %}
	/* Process time events */
	static int processTimeEvents(aeEventLoop *eventLoop) {
	    int processed = 0;
	    aeTimeEvent *te;
	    long long maxId;
	    time_t now = time(NULL);
	
	    /* If the system clock is moved to the future, and then set back to the
	     * right value, time events may be delayed in a random way. Often this
	     * means that scheduled operations will not be performed soon enough.
	     *
	     * Here we try to detect system clock skews, and force all the time
	     * events to be processed ASAP when this happens: the idea is that
	     * processing events earlier is less dangerous than delaying them
	     * indefinitely, and practice suggests it is. */
	    if (now < eventLoop->lastTime) {
	        te = eventLoop->timeEventHead;
	        while(te) {
	            te->when_sec = 0;
	            te = te->next;
	        }
	    }
	    eventLoop->lastTime = now;
	
	    te = eventLoop->timeEventHead;
	    maxId = eventLoop->timeEventNextId-1;
	    while(te) {
	        long now_sec, now_ms;
	        long long id;
	
	        if (te->id > maxId) {
	            te = te->next;
	            continue;
	        }
	        aeGetTime(&now_sec, &now_ms);
			/* 判断时间事件是否到达 */
	        if (now_sec > te->when_sec ||
	            (now_sec == te->when_sec && now_ms >= te->when_ms))
	        {
	            int retval;
	
				/* 调用时间事件处理函数 */
	            id = te->id;
	            retval = te->timeProc(eventLoop, id, te->clientData);
	            processed++;
	            /* After an event is processed our time event list may
	             * no longer be the same, so we restart from head.
	             * Still we make sure to don't process events registered
	             * by event handlers itself in order to don't loop forever.
	             * To do so we saved the max ID we want to handle.
	             *
	             * FUTURE OPTIMIZATIONS:
	             * Note that this is NOT great algorithmically. Redis uses
	             * a single time event so it's not a problem but the right
	             * way to do this is to add the new elements on head, and
	             * to flag deleted elements in a special way for later
	             * deletion (putting references to the nodes to delete into
	             * another linked list). */
	            if (retval != AE_NOMORE) {
	                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
	            } else {
	                aeDeleteTimeEvent(eventLoop, id);
	            }
	            te = eventLoop->timeEventHead;
	        } else {
	            te = te->next;
	        }
	    }
	    return processed;
	}
{% endhighlight %}

如果当前时间 now 小于 `eventloop->lastTime`，那么

	if (now < eventLoop->lastTime) {
		te = eventLoop->timeEventHead;
	    while(te) {
	    	te->when_sec = 0;
	        te = te->next;
	    }
	}

redis 会处理整个时间链表中的所有时间事件。

一个时间事件时定时事件还是周期性事件时根据时间处理函数的返回值来判断的： 

* 如果返回值为 `AE_NOMORE`，改时间为定时事件，该事件在达到处理之后，将会被从时间事件链表中删除，不在执行
* 如果返回值是非 `AE_NOMORE` 的值，那么该事件是周期性事件，更新时间的 when_sec 和 when_ms 的值，等到下一次事件到达时继续执行。

**目前版本的 redis 只是用周期性事件，还没有使用定时事件。**

### 时间事件的使用 servCron 事件
在redis 服务器初始化时，会创建时间事件

    /* Create the serverCron() time event, that's our main way to process
     * background operations. */
    if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        redisPanic("Can't create the serverCron time event.");
        exit(1);
    }

该时间事件的处理函数为 `serverCron`。
# 服务器中的 client 状态
Redis 服务器负责与多个客户端建立网络连接，处理客户端发送的命令请求，在数据库中保存客户端执行命令所产生的数据，并通过资源管理来维持服务器自身的运转。<br>
对每个与服务器连接的客户端，服务器都为这些客户端建立了相应的结构，用于保存客户端的状态信息，以及执行相关功能时需要用到的数据结构， `redis.h/redisClient`

{% highlight ruby %}
	/* With multiplexing we need to take per-client state.
 	 * Clients are taken in a linked list. */
	typedef struct redisClient {
	    uint64_t id;            /* Client incremental unique ID. */
	    int fd;					/* socket file descriptor */
	    redisDb *db;
	    int dictid;
	    robj *name;             /* As set by CLIENT SETNAME */
	    sds querybuf;
	    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size */
	    int argc;
	    robj **argv;
	    struct redisCommand *cmd, *lastcmd;
	    int reqtype;
	    int multibulklen;       /* number of multi bulk arguments left to read */
	    long bulklen;           /* length of bulk argument in multi bulk request */
	    list *reply;
	    unsigned long reply_bytes; /* Tot bytes of objects in reply list */
	    int sentlen;            /* Amount of bytes already sent in the current
	                               buffer or object being sent. */
	    time_t ctime;           /* Client creation time */
	    time_t lastinteraction; /* time of the last interaction, used for timeout */
	    time_t obuf_soft_limit_reached_time;
	    int flags;              /* REDIS_SLAVE | REDIS_MONITOR | REDIS_MULTI ... */
	    int authenticated;      /* when requirepass is non-NULL */
	    int replstate;          /* replication state if this is a slave */
	    int repl_put_online_on_ack; /* Install slave write handler on ACK. */
	    int repldbfd;           /* replication DB file descriptor */
	    off_t repldboff;        /* replication DB file offset */
	    off_t repldbsize;       /* replication DB file size */
	    sds replpreamble;       /* replication DB preamble. */
	    long long reploff;      /* replication offset if this is our master */
	    long long repl_ack_off; /* replication ack offset, if this is a slave */
	    long long repl_ack_time;/* replication ack time, if this is a slave */
	    long long psync_initial_offset; /* FULLRESYNC reply offset other slaves
	                                       copying this slave output buffer
	                                       should use. */
	    char replrunid[REDIS_RUN_ID_SIZE+1]; /* master run id if this is a master */
	    int slave_listening_port; /* As configured with: SLAVECONF listening-port */
	    int slave_capa;         /* Slave capabilities: SLAVE_CAPA_* bitwise OR. */
	    multiState mstate;      /* MULTI/EXEC state */
	    int btype;              /* Type of blocking op if REDIS_BLOCKED. */
	    blockingState bpop;     /* blocking state */
	    long long woff;         /* Last write global replication offset. */
	    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
	    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */
	    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
	    sds peerid;             /* Cached peer ID. */
	
	    /* Response buffer */
	    int bufpos;
	    char buf[REDIS_REPLY_CHUNK_BYTES];	/* 16K output buffer, soft limit */
	} redisClient;
{% endhighlight %}

1. `fd` ： 套接字文件描述符 <br>
2. `name` ： 客户端的名字，是一个 redisObject 对象，redisStringObject 对象 <br>
3. `db` ： 客户端使用的数据库的指针 <br>
4. `argc, argv, cmd, lastcmd` ： 客户端命令参数及指向执行命令的函数指针 <br>
5. `flags` ： 客户端的标识，记录了客户端的角色以及目前客户端的状态 <br>
6. `querybuf, buf` ： 输入和输出缓冲区 <br>
7. `ctime` ： 客户端创建时间 <br>
8. `lastinteraction` ： 客户端与服务器最后一次通信时间 <br>
9. `obuf_soft_limit_reached_time` ： 客户端输出缓冲区大小超出软性限制的时间
10. ……

## 客户端中的几个重要属性
### 标志 flags
flags 属性的值可以是单个标志：
	
	flags = <flag>

也可以是多个标志的二进制：

	flags = <flag1> | <flag2> | ...

redis 中客户端标志的宏定义如下所示

	/* Client flags */
	#define REDIS_SLAVE (1<<0)   /* This client is a slave server */
	#define REDIS_MASTER (1<<1)  /* This client is a master server */
	#define REDIS_MONITOR (1<<2) /* This client is a slave monitor, see MONITOR */
	#define REDIS_MULTI (1<<3)   /* This client is in a MULTI context */
	#define REDIS_BLOCKED (1<<4) /* The client is waiting in a blocking operation */
	#define REDIS_DIRTY_CAS (1<<5) /* Watched keys modified. EXEC will fail. */
	#define REDIS_CLOSE_AFTER_REPLY (1<<6) /* Close after writing entire reply. */
	#define REDIS_UNBLOCKED (1<<7) /* This client was unblocked and is stored in
	                                  server.unblocked_clients */
	#define REDIS_LUA_CLIENT (1<<8) /* This is a non connected client used by Lua ，表示客户端是专门用于处理lua脚本中包含 redis 命令的伪客户端 */
	#define REDIS_ASKING (1<<9)     /* Client issued the ASKING command */
	#define REDIS_CLOSE_ASAP (1<<10)/* Close this client ASAP */
	#define REDIS_UNIX_SOCKET (1<<11) /* Client connected via Unix domain socket */
	#define REDIS_DIRTY_EXEC (1<<12)  /* EXEC will fail for errors while queueing */
	#define REDIS_MASTER_FORCE_REPLY (1<<13)  /* Queue replies even if is master */
	#define REDIS_FORCE_AOF (1<<14)   /* Force AOF propagation of current cmd. */
	#define REDIS_FORCE_REPL (1<<15)  /* Force replication of current cmd. */
	#define REDIS_PRE_PSYNC (1<<16)   /* Instance don't understand PSYNC. */
	#define REDIS_READONLY (1<<17)    /* Cluster client is in read-only state. */
	#define REDIS_PUBSUB (1<<18)      /* Client is in Pub/Sub mode. */

### 输入缓冲区 querybuf
redis 客户端状态信息中的输入缓冲区 querybuf 用于保存客户端发送的命令请求， `readQueryFromClient` 这个函数就是读取客户端发送的命令请求并保存在 querybuf 中，该缓冲区的最大大小为 1GB，当超出这个值时，服务器将关闭这个客户端。

### 命令与命令参数 argc, argv
argc 表示客户端发送的命令参数的个数， argv 是一个 `redisObject` 结构体的数组，每一个参数就是一个 `redisObject` 类型的变量。当服务器读取完客户端发送的命令请求之后，通过 `Networking.c/processInlineBuffer` 和 `Networking.c/processMultiBulkBuffer` 这两个函数，将 querybuf 中的内容解析后，存放在 argv 中，argc 保存的是参数的个数。

### 命令实现函数 cmd, lastcmd
当参数解析存放在 argv 中后，redis服务器会通过 argv[0] 查找命令处理函数，在 `redis.c/processCommand` 中

	c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
在 `redis.c/initServerConfig` 中调用 `populateCommandTable` 函数，初始化  `server.commands` 字典，通过命令名称，在字典中查找对应的命令实现函数。

	struct redisCommand redisCommandTable[] = {
	    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
	    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
	    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
	    {"setex",setexCommand,4,"wm",0,NULL,1,1,1,0,0},
	    {"psetex",psetexCommand,4,"wm",0,NULL,1,1,1,0,0},
	    {"append",appendCommand,3,"wm",0,NULL,1,1,1,0,0},
	    {"strlen",strlenCommand,2,"rF",0,NULL,1,1,1,0,0},
	    {"del",delCommand,-2,"w",0,NULL,1,-1,1,0,0},
	    {"exists",existsCommand,-2,"rF",0,NULL,1,-1,1,0,0},
	    {"setbit",setbitCommand,4,"wm",0,NULL,1,1,1,0,0},
	    {"getbit",getbitCommand,3,"rF",0,NULL,1,1,1,0,0},
	    {"setrange",setrangeCommand,4,"wm",0,NULL,1,1,1,0,0},
	    ...
	};

### 输出缓冲区

	/* Response buffer */
	    int bufpos;
	    char buf[REDIS_REPLY_CHUNK_BYTES];	/* 16K output buffer */
		...
		list *reply;

服务器执行命令结果会保存在输出缓冲区，每一个客户端都会有两个缓冲区，一个固定大小的缓冲区和一个可变大小的缓冲区。

* 固定大小的缓冲区用于保存长度较小的结果，比如 OK、整数值、错误回复、简短的字符串值等。
* 可变大小的缓冲区用于保存那些长度比较大的结果，比如包含了很多元素的集合或者一个非常大的字符串值等。

在固定大小的缓冲区中，buf 长度最大为 16K，bufpos 为实际使用的字节数。

可变大小的缓冲区由 reply 链表组成，这是一个双向链表。链表长度不受 16KB 的限制。

### 验证 authenticated
客户端的 `authenticated` 属性，用于记录客户端是否通过验证。如果值为0，表示未通过验证；如果为1，表示通过。

当 `authenticated` 的值为0时，客户端发送的命令除了 `AUTH` 之外，其余的所有命令将都会被服务器拒绝执行。

`authenticated` 属性只有在服务器启用了身份验证功能时使用，在 `redis.config` 配置文件中通过设置 `requirepass` 选项可以设置该功能。如果没有启动身份验证功能，及时 `authenticated` 的值为0，服务器也不会拒绝客户端的命令请求。

# 服务器实现的细节 ( redis.c/main )
redis 服务器启动时，需要做很多准备工作

1. 设置编码 `setlocale(LC_COLLATE,"");` <br>
2. 设置线程安全模式 `zmalloc_enable_thread_safeness();` <br>
3. 设置 OOM 异常处理方法 `zmalloc_set_oom_handler(redisOutOfMemoryHandler);` <br>
4. 设置哈希种子 `dictSetHashFunctionSeed(tv.tv_sec^tv.tv_usec^getpid())` <br>
5. 检查服务器是否是以 Sentinel Mode 的方式启动 `checkForSentinelMode(argc,argv)` <br>
6. 读取 redis.config 配置文件，初始化服务器配置 `initServerConfig()` <br>
7. 初始化服务器参数 `initServer()` <br>
8. 启动服务器守护进程模式 `daemonize()` <br>
9. 创建 pid 文件 `createPidFile()` <br>
10. 进入主循环 `aeMain()`

## 初始化服务器
server 是一个全局变量，在 `redis.c` 中定义

	/* Global vars */
	struct redisServer server; /* server global state */

### 初始化服务器状态结构
在 `redis.c/initServerConfig()` 函数中，对 `server` 变量进行了初始化

	void initServerConfig(void) {
	    int j;
	
	    getRandomHexChars(server.runid,REDIS_RUN_ID_SIZE);	//get redis "Run ID" by SHA algorithm, to keep every redis "Run ID" are different
	    server.configfile = NULL;	//配置文件
	    server.hz = REDIS_DEFAULT_HZ;	//服务器频率
	    server.runid[REDIS_RUN_ID_SIZE] = '\0';
	    server.arch_bits = (sizeof(long) == 8) ? 64 : 32;	//服务器运行架构
	    server.port = REDIS_SERVERPORT;		//默认端口，一般是6379
	    server.tcp_backlog = REDIS_TCP_BACKLOG;	//默认监听队列长度
		...
		server.lruclock = getLRUClock();	//初始化LRU时钟
		...
		populateCommandTable();		//创建命令表
		..
	}

`getRandomHexChars` 函数是通过 SHA1 算法获取 server 的 runid，摆正 runid 的唯一性，在 redis 注释中也有如下说明

	/* Generate the Redis "Run ID", a SHA1-sized random number that identifies a
	 * given execution of Redis, so that if you are talking with an instance
	 * having run_id == A, and you reconnect and it has run_id == B, you can be
	 * sure that it is either a different instance or it was restarted. */

`initServerConfig` 函数只创建了服务器状态的一些基本属性参数，比如整数、浮点数和字符串属性，但是对数据库、Lua环境、共享对象、慢查询日志这些数据结构的初始化并没有创建，这些将在后面实现。

### 载入配置选项
redis 服务器启动时，一般会指定配置文件，如果没有指定配置文件参数，系统谁使用默认的配置文件，比如 `redis.config`

服务器通过 `loadServerConfig` 函数加载配置文件

{% highlight ruby %}
	/* Load the server configuration from the specified filename.
	 * The function appends the additional configuration directives stored
	 * in the 'options' string to the config file before loading.
	 *
	 * Both filename and options can be NULL, in such a case are considered
	 * empty. This way loadServerConfig can be used to just load a file or
	 * just load a string. */
	void loadServerConfig(char *filename, char *options) {
	    sds config = sdsempty();
	    char buf[REDIS_CONFIGLINE_MAX+1];
	
	    /* Load the file content */
	    if (filename) {
	        FILE *fp;
	
	        if (filename[0] == '-' && filename[1] == '\0') {
	            fp = stdin;
	        } else {
	            if ((fp = fopen(filename,"r")) == NULL) {
	                redisLog(REDIS_WARNING,
	                    "Fatal error, can't open config file '%s'", filename);
	                exit(1);
	            }
	        }
	        while(fgets(buf,REDIS_CONFIGLINE_MAX+1,fp) != NULL)
	            config = sdscat(config,buf);
	        if (fp != stdin) fclose(fp);
	    }
	    /* Append the additional options */
	    if (options) {
	        config = sdscat(config,"\n");
	        config = sdscat(config,options);
	    }
	    loadServerConfigFromString(config);
	    sdsfree(config);
	}
{% endhighlight %}
`loadServerConfig` 函数将配置文件全部加载到 config 变量中，整个文件的参数都加载到 config 字符串变量中，此时，config 是一个很长很长的字符串变量，然后通过 `loadServerConfigFromSrting` 函数，将 config 进行分割，并对 server 中的相关参数进行赋值。

{% highlight ruby %}
	void loadServerConfigFromString(char *config) {
	    char *err = NULL;
	    int linenum = 0, totlines, i;
	    int slaveof_linenum = 0;
	    sds *lines;
	
	    lines = sdssplitlen(config,strlen(config),"\n",1,&totlines);
	
	    for (i = 0; i < totlines; i++) {
	        sds *argv;
	        int argc;
	
	        linenum = i+1;
	        lines[i] = sdstrim(lines[i]," \t\r\n");
	
	        /* Skip comments and blank lines */
	        if (lines[i][0] == '#' || lines[i][0] == '\0') continue;
	
	        /* Split into arguments */
	        argv = sdssplitargs(lines[i],&argc);
	        if (argv == NULL) {
	            err = "Unbalanced quotes in configuration line";
	            goto loaderr;
	        }
	
	        /* Skip this line if the resulting command vector is empty. */
	        if (argc == 0) {
	            sdsfreesplitres(argv,argc);
	            continue;
	        }
	        sdstolower(argv[0]);
	
	        /* Execute config directives */
	        if (!strcasecmp(argv[0],"timeout") && argc == 2) {
	            server.maxidletime = atoi(argv[1]);
	            if (server.maxidletime < 0) {
	                err = "Invalid timeout value"; goto loaderr;
	            }
	        } else if (!strcasecmp(argv[0],"tcp-keepalive") && argc == 2) {
	            server.tcpkeepalive = atoi(argv[1]);
	            if (server.tcpkeepalive < 0) {
	                err = "Invalid tcp-keepalive value"; goto loaderr;
	            }
			...
	}
{% endhighlight %}

服务器在载入用户指定的配置选项，并对 server 状态进行更新之后，服务器就进入初始化第三个阶段 -- 初始化服务器数据结构。

### 服务器的守护进程实现
大家都知道，如何实现一个守护进程，首先需要了解守护进程的特征。

1. 大多数守护进程都是以 root 超级用户权限运行。 <br>
2. 所有的守护进程都没有控制终端，ps 查看的结果中终端名设置为 ? <br>
3. 内核守护进程以无控制终端方式运行，而用户层守护进程无控制终端可能是调用 setsid 的结果。<br>
4. 大多数用户层进程都是进程组的组长进程以及会话的首进程，同时也是这些进程组和会话中的唯一进程。 <br>
5. 用户层守护进程的父进程是 init 进程。

那么，根据以上特征，按照一定的规则就能创建守护进程，这里所说的一般都是用户层的守护进程。

(1) 首先需要做的就是 fork 创建一个进程，然后使父进程退出(子进程成为孤儿进程)，此时，如果是在 terminal 上启动的，子进程继承父进程的属性，会继承父进程的 umask 掩码、进程组、控制终端属性等。

(2) 父进程退出之后，使用 setsid ，新创建一个会话 session。<br>
　　一个会话可以包含一个或多个进程组，一个进程组可以包含一个或多个进程。这些进程组可共享一个控制终端，所以该会话与控制终端相联系。控制终端与会话是一一对应的。因为父进程创建子进程，所以该子进程不可能是父进程所在进程组的组长和会话组长，使用 setsid 创建一个新的会话，此时，该进程成为这个会话的唯一进程，也是这个会话中进程组组长。 <br>
　　setsid 函数在进程时进程组组长时会执行失败。如果执行成功，那么，因为会话与控制终端是一一对应的，此时，该进程将摆脱父进程的影响，存在一个新的进程组和会话中，并且与控制终端不相关。

(3) 使用 `umask (0)`，将文件掩码清除，继承自父进程的掩码，可能会被设置为拒绝某些权限。

(4) 将当前工作目录更改为根目录。从父进程继承来的工作目录可能挂载某一个文件系统中。因为守护进程通常是在系统引导之前一直存在的，所以如果守护进程的工作目录挂载在某一个文件系统中，该文件系统将不能被卸载。

(5) 关闭不再需要的文件描述符。一般将 STDIN、STDOUT、STDERR都重定向到 `/dev/null` 空洞文件中，然后在关闭 0,1,2 文件描述符。因为守护进程不与终端设备相关联，所以输出无处显示，也无法从交互式用户那里接收输入。

	struct rlimit rl;
	getrlimit (RLIMIT_NOFILE, &rl);
	
	int j;
	for (j=0; i<rl.rlim_max; i++) {
		close (i);
	}
	
	int fd;
	if ((fd = open("/dev/null", O_RDWR, 0)) != -1) {
		dup2(fd, STDIN_FILENO);
		dup2(fd, STDOUT_FILENO);
		dup2(fd, STDERR_FILENO);
		if (fd > STDERR_FILENO) close(fd);
	}

redis 的 `daemonize()` 函数的实现如下所示，实现 redis 的守护进程

	void daemonize(void) {
	    int fd;
	
	    if (fork() != 0) exit(0); /* parent exits */
	    setsid(); /* create a new session */
	
	    /* Every output goes to /dev/null. If Redis is daemonized but
	     * the 'logfile' is set to 'stdout' in the configuration file
	     * it will not log at all. */
	    if ((fd = open("/dev/null", O_RDWR, 0)) != -1) {
	        dup2(fd, STDIN_FILENO);
	        dup2(fd, STDOUT_FILENO);
	        dup2(fd, STDERR_FILENO);
	        if (fd > STDERR_FILENO) close(fd);
	    }
	}

> 参考： <br>
> 《Unix 高级环境编程（第3版）》，第13章，守护进程。
### 初始化服务器数据结构
在 `initServerConfig` 函数中，程序只初始化了服务器命令表这一个数据结构，其他的数据结构在 `initServer` 函数中进行初始化。比如：

* server.clients 链表，这是一个服务器端维护客户端状态的链表，记录了客户端的参数、命令执行函数、与服务器最近交互的时间等信息。
* server.db 数组，包含了服务器中的所有数据库，一般默认是 16 个数据库。
* server.lua 用户执行 Lua 脚本的 Lua 环境
* server.slowlog 用于保存慢查询日志

**问题：服务器为什么在 `initServerConfig` 初始化状态结构，并加载完配置文件后才初始化这些数据结构呢？**

这是因为，用户可以在配置文件中制定相关的配置选项参数，服务器必须先载入用户指定的配置选项，否则，当用户修改配置文件参数时，服务器就需要重新调整和修改已经创建好的数据结构。

当然，`initServer` 函数还做了一些其他的操作：

- 为服务器设置进程信号处理器 `setupSignalHandlers()`
- 创建共享对象 `createSharedObjects()`，共享对象是一个全局变量，在 `redis.c` 中申明

	`struct sharedObjectsStruct shared;` <br>
大部分都是一些能够共享的字符串类型的对象，比如错误消息等。

- 打开服务器的监听端口 `Listen()`，并创建文件事件，为套接字关联连接应答处理器，等待服务器正式运行时接收客户端的连接。
- 创建时间事件，关联 `serverCron` 函数
- 如果AOF持久化功能打开，那么打开现有的AOF文件，如果AOF文件不存在，那么创建并打开一个新的AOF文件，为AOF写入做好准备。

{% highlight ruby %}
    /* Open the AOF file if needed. */
    if (server.aof_state == REDIS_AOF_ON) {
        server.aof_fd = open(server.aof_filename,
                               O_WRONLY|O_APPEND|O_CREAT,0644);
        if (server.aof_fd == -1) {
            redisLog(REDIS_WARNING, "Can't open the append-only file: %s",
                strerror(errno));
            exit(1);
        }
    }
{% endhighlight %}

- 初始化服务器后台 `I/O` 模块（bio），为 `I/O` 操作做好准备。 `bioInit()`

## 还原数据库状态
在完成了 server 的一系列初始化之后，服务器需要载入 AOF 文件或者 RDB 文件来还原数据库的状态。但是，在载入这些文件之前，服务器还需要检查一下系统参数是否正常。

#### 检查系统允许的套接字监听队列长度的最大值
	/* Check that server.tcp_backlog can be actually enforced in Linux according
	 * to the value of /proc/sys/net/core/somaxconn, or warn about it. */
	void checkTcpBacklogSettings(void) {
	#ifdef HAVE_PROC_SOMAXCONN
	    FILE *fp = fopen("/proc/sys/net/core/somaxconn","r");
	    char buf[1024];
	    if (!fp) return;
	    if (fgets(buf,sizeof(buf),fp) != NULL) {
	        int somaxconn = atoi(buf);
	        if (somaxconn > 0 && somaxconn < server.tcp_backlog) {
	            redisLog(REDIS_WARNING,"WARNING: The TCP backlog setting of %d cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of %d.", server.tcp_backlog, somaxconn);
	        }
	    }
	    fclose(fp);
	#endif
	}

对于一个TCP连接，Server 与 Client 需要通过三次握手来建立网络连接。当三次握手成功后,我们可以看到端口的状态由 LISTEN 转变为 ESTABLISHED。接着这条链路上就可以开始传送数据了。

每一个处于监听(Listen)状态的端口,都有自己的监听队列.监听队列的长度,与如下两方面有关:

- somaxconn参数，在 rhel 中，`/proc/sys/net/core/somaxconn`
- 使用该端口的程序中 `listen(int sockfd, int backlog)` 函数.
#### 检查内存状态

	#ifdef __linux__
	int linuxOvercommitMemoryValue(void) {
	    FILE *fp = fopen("/proc/sys/vm/overcommit_memory","r");
	    char buf[64];
	
	    if (!fp) return -1;
	    if (fgets(buf,64,fp) == NULL) {
	        fclose(fp);
	        return -1;
	    }
	    fclose(fp);
	
	    return atoi(buf);
	}

`overcommit_memory` 文件指定了内核针对内存分配的策略，其值可以是0、1、2。 

* 0， 表示内核将检查是否有足够的可用内存供应用进程使用；如果有足够的可用内存，内存申请允许；否则，内存申请失败，并把错误返回给应用进程。 
* 1， 表示内核允许分配所有的物理内存，而不管当前的内存状态如何。
* 2， 表示内核允许分配超过所有物理内存和交换空间总和的内存 

__什么是Overcommit和OOM__<br>
　　Linux对大部分申请内存的请求都回复"yes"，以便能跑更多更大的程序。因为申请内存后，并不会马上使用内存。这种技术叫做Overcommit。当linux发现内存不足时，会发生OOM killer(OOM=out-of-memory)。它会选择杀死一些进程(用户态进程，不是内核线程)，以便释放内存。 <br>
　　当 `oom-killer`发生时，linux会选择杀死哪些进程？选择进程的函数是`oom_badness` 函数(在`mm/oom_kill.c`中)，该函数会计算每个进程的点数(0~1000)。点数越高，这个进程越有可能被杀死。每个进程的点数跟`oom_score_adj` 有关，而且 `oom_score_adj` 可以被设置(-1000最低，1000最高)。

__当 redis 中因为 `overcommit_memory` 系统参数出现问题时，会出现如下的日志信息__ <br>

	17 Mar 13:18:02.207 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.

	__解决办法__ <br>
在 root 权限下，修改内核参数

* 编辑 /etc/sysctl.conf ，改 `vm.overcommit_memory=1` ，然后 `sysctl -p`  使配置文件生效
* `sysctl vm.overcommit_memory=1`
* `echo 1 > /proc/sys/vm/overcommit_memory`


__在 redis 中，需要查看系统是否支持 THP__，即 `Transparent Huge Page`(透明巨页)

	#ifdef __linux__
	/* Returns 1 if Transparent Huge Pages support is enabled in the kernel.
	 * Otherwise (or if we are unable to check) 0 is returned. */
	/* my /sys/kernel/mm/transparent_hugepage/enabled is "[always] never", so THP is set on, if file content is [never], THP is set off */
	int THPIsEnabled(void) {
	    char buf[1024];
	
	    FILE *fp = fopen("/sys/kernel/mm/transparent_hugepage/enabled","r");
	    if (!fp) return 0;
	    if (fgets(buf,sizeof(buf),fp) == NULL) {
	        fclose(fp);
	        return 0;
	    }
	    fclose(fp);
	    return (strstr(buf,"[never]") == NULL) ? 1 : 0;
	}
	#endif

　　一般而言，内存管理的最小块级单位叫做 page ，一个 page 是 4096 bytes，1M 的内存会有256个 page，1GB的话就会有256,000个 page。CPU 通过内置的内存管理单元维护着 page 表记录。 <br>
　　现代的硬件内存管理单元最多只支持数百到上千的 page 表记录，并且，对于数百万 page 表记录的维护算法必将与目前的数百条记录的维护算法大不相同才能保证性能，目前的解决办法是，如果一个程序所需内存page数量超过了内存管理单元的处理大小，操作系统会采用软件管理的内存管理单元，但这会使程序运行的速度变慢。 <br>
　　从redhat 6（centos，sl，ol）开始，操作系统开始支持 Huge Pages，也就是大页。 <br>
　　简单来说， Huge Pages就是大小为 2M 到 1GB 的内存 page，主要用于管理数千兆的内存，比如 1GB 的 page 对于 1TB 的内存来说是相对比较合适的。 <br>
　　THP（Transparent Huge Pages）是一个使管理 Huge Pages 自动化的抽象层。使用透明巨页内存的好处： <br>

1. 可以使用swap，内存页默认是2M大小，需要使用swap的时候，内存被分割为4k大小 <br>
2. 对用户透明，不需要用户做特殊配置 <br>
3. 不需要root权限 <br>
4. 不需要依某种库文件 <br>

> **参考**： <br>
1. 有关 linux 下 redis overcommit_memory 的问题 [http://blog.csdn.net/whycold/article/details/21388455] <br>
2. Transparent Huge Pages 相关概念及对 mysql 的影响 [https://my.oschina.net/llzx373/blog/226446] <br>
3. 透明大页介绍 [http://www.cnblogs.com/kerrycode/archive/2015/07/23/4670931.html] <br>

#### 加载 AOF 或者 RDB 文件
	/* Function called at startup to load RDB or AOF file in memory. */
	void loadDataFromDisk(void) {
	    long long start = ustime();	//get current time as seconds
	    if (server.aof_state == REDIS_AOF_ON) {
	        if (loadAppendOnlyFile(server.aof_filename) == REDIS_OK)
	            redisLog(REDIS_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
	    } else {
	        if (rdbLoad(server.rdb_filename) == REDIS_OK) {
	            redisLog(REDIS_NOTICE,"DB loaded from disk: %.3f seconds",
	                (float)(ustime()-start)/1000000);
	        } else if (errno != ENOENT) {
	            redisLog(REDIS_WARNING,"Fatal error loading the DB: %s. Exiting.",strerror(errno));
	            exit(1);
	        }
	    }
	}

如果服务器启用了 AOF 持久化功能，`server.aof_state == REDIS_AOF_ON`，服务器使用 AOF 文件来还原数据库状态；否则，服务器使用 RDB 文件来还原数据库状态。

## 执行事件循环
	void aeMain(aeEventLoop *eventLoop) {
	    eventLoop->stop = 0;
	    while (!eventLoop->stop) {
	        if (eventLoop->beforesleep != NULL)
	            eventLoop->beforesleep(eventLoop);
	        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
	    }
	}

事件循环，处理文件事件和时间事件。

## 服务器接收回复客户端的详细经过
一个命令从客户端发送到服务器，再由服务器接收执行和回复的经过，需要客户端和服务器完成一系列的操作。

### 命令请求的执行过程
![send and reply](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/redis_analysis/send_and_reply.png) <br>
加入客户端发送 `SET KEY REDIS` 命令给服务器到获得回复 OK 期间，需要共同完成以下操作： <br>
1) 客户端向服务器发送命令请求 <br>
2) 服务器接收到客户端发送的命令请求，执行操作，并在数据库中设置，操作成功后产生命令回复OK <br>
3) 服务器将命令结果OK发送给客户端 <br>
4) 客户端接收到命令回复OK，打印给用户

#### 发送命令请求
在前面的章节《redis源码分析 -- cs结构分析之客户端》[http://blog.csdn.net/honglicu123/article/details/53169843]中已经介绍了客户端发送命令到服务器的细节，用户在客户端键入命令，发送到服务器时，是按照 redis 协议格式发送的。

#### 读取命令请求
当服务器初始化成功后，创建文件事件，将套接字与连接请求处理器关联，当客户端与服务器连接之后，就会创建文件事件，将套接字与命令请求处理器连接，客户端向服务器发送命令请求，触发该事件，引发命令请求处理器处理，接收客户端的命令。

1) 读取套接字中协议格式的命令请求，并保存到客户端状态的输入缓冲区中 c->querybuf

{% highlight ruby %}
	void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
		...
		qblen = sdslen(c->querybuf);
	    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
	    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
	    nread = read(fd, c->querybuf+qblen, readlen);
		...
		if (nread) {
	        sdsIncrLen(c->querybuf,nread);
	        c->lastinteraction = server.unixtime;
	        if (c->flags & REDIS_MASTER) c->reploff += nread;
	        server.stat_net_input_bytes += nread;
	    }
		...
	}
{% endhighlight %}

2) 对输入缓冲区中的命令进行解析，将参数和参数个数保存在客户端状态的 argc 和 argv 中，`networking.c/processInlineBuffer` 和 `networking.c/processMultibulkBuffer` 就是完成这个操作。将redis协议格式的命令请求解析之后，每一个命令参数都生成一个 redisStringObject 类型的结构，保存在 argv 数组中。比如 `SET NAME REDIS` ，在客户端状态结构中将如下所示的形式存储

![redis client get command](https://github.com/small-cat/small-cat.github.io/raw/master/_pics/redis_analysis/redisClient_getCommand.png) <br>
3) 调用命令执行函数，执行命令。

**问题：命令时如何执行的呢**

#### 命令执行过程
**一、 查找命令实现函数** <br>
在服务器初始化 `initServerConfig` 函数中，

{% highlight ruby %}
	/* Command table -- we initiialize it here as it is part of the
	     * initial configuration, since command names may be changed via
	     * redis.conf using the rename-command directive. */
	    server.commands = dictCreate(&commandTableDictType,NULL);
	    server.orig_commands = dictCreate(&commandTableDictType,NULL);
	    populateCommandTable();
	    server.delCommand = lookupCommandByCString("del");
	    server.multiCommand = lookupCommandByCString("multi");
	    server.lpushCommand = lookupCommandByCString("lpush");
	    server.lpopCommand = lookupCommandByCString("lpop");
	    server.rpopCommand = lookupCommandByCString("rpop");
{% endhighlight %}

对命令表做了初始化，创建了命令表字典，在上面 服务器中的客户端状态 小节中有所描述。

当需要执行命令时，首先根据客户端状态中解析出的命令参数 argv[0] 在命令表字典中查找命令实现函数

	c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
	
c->cmd 和 c->lastcmd 是 redisCommand 结构的指针

	struct redisCommand {
	    char *name;
	    redisCommandProc *proc;
	    int arity;
	    char *sflags; /* Flags as string representation, one char per flag. */
	    int flags;    /* The actual flags, obtained from the 'sflags' field. */
	    /* Use a function to determine keys arguments in a command line.
	     * Used for Redis Cluster redirect. */
	    redisGetKeysProc *getkeys_proc;
	    /* What keys should be loaded in background when calling this command? */
	    int firstkey; /* The first argument that's a key (0 = no keys) */
	    int lastkey;  /* The last argument that's a key */
	    int keystep;  /* The step between first and last key */
	    long long microseconds, calls;
	};

name ：是命令的名称，比如 “SET” <br>
proc ：是命令实现函数指针，命令SET的命令实现函数为 `setCommand`。<br>
arity：命令参数的个数，用于检查命令请求的格式是否正确。如果是负值 -N，表明这个命令的参数个数大于等于N，如果是正数，就表明参数个数为N <br>
sflags：字符串形式的标识，比如 "wrm"，这个在初始化命令字典表示，有定义

{% highlight ruby %}
	/* Populates the Redis Command Table starting from the hard coded list
	 * we have on top of redis.c file. */
	void populateCommandTable(void) {
	    int j;
	    int numcommands = sizeof(redisCommandTable)/sizeof(struct redisCommand);
	
	    for (j = 0; j < numcommands; j++) {
	        struct redisCommand *c = redisCommandTable+j;
	        char *f = c->sflags;
	        int retval1, retval2;
	
	        while(*f != '\0') {
	            switch(*f) {
	            case 'w': c->flags |= REDIS_CMD_WRITE; break;
	            case 'r': c->flags |= REDIS_CMD_READONLY; break;
	            case 'm': c->flags |= REDIS_CMD_DENYOOM; break;
	            case 'a': c->flags |= REDIS_CMD_ADMIN; break;
	            case 'p': c->flags |= REDIS_CMD_PUBSUB; break;
	            case 's': c->flags |= REDIS_CMD_NOSCRIPT; break;
	            case 'R': c->flags |= REDIS_CMD_RANDOM; break;
	            case 'S': c->flags |= REDIS_CMD_SORT_FOR_SCRIPT; break;
	            case 'l': c->flags |= REDIS_CMD_LOADING; break;
	            case 't': c->flags |= REDIS_CMD_STALE; break;
	            case 'M': c->flags |= REDIS_CMD_SKIP_MONITOR; break;
	            case 'k': c->flags |= REDIS_CMD_ASKING; break;
	            case 'F': c->flags |= REDIS_CMD_FAST; break;
	            default: redisPanic("Unsupported command flag"); break;
	            }
	            f++;
	        }
	
	        retval1 = dictAdd(server.commands, sdsnew(c->name), c);
	        /* Populate an additional dictionary that will be unaffected
	         * by rename-command statements in redis.conf. */
	        retval2 = dictAdd(server.orig_commands, sdsnew(c->name), c);
	        redisAssert(retval1 == DICT_OK && retval2 == DICT_OK);
	    }
	}
{% endhighlight %}

flags：是对 sflags 分析得出的二进制标识 <br>
calls：记录服务器执行该命令的次数 <br>
milliseconds：记录服务器执行该命令所耗费总时长

**二、命令执行前的检查工作** <br>
1. 检查命令实现函数是否查找成功，如果 cmd 为NULL，说明没有找到该命令的实现函数，返回客户端一个错误 “unknown command” <br>
2. 根据 cmd 的 arity 属性，检查命令的参数格式是否正确，如果不正确，返回客户端错误信息 "wrong number of arguments for XX command" <br>
3. 检查服务器是否启用 `requirepass`，如果启用检查客户端是否通过身份验证，未通过验证的客户端只能执行 AUTH 命令，其他命令，服务器将返回客户端一个错误信息 “-NOAUTH Authentication required.\r\n” <br>
4. 如果服务器打开了 `maxmemory` 功能，在执行命令之前，先检查内存占用情况，在需要的情况下，会回收一部分内存。如果执行失败，将返回错误 “-OOM command not allowed when used memory > 'maxmemory'.\r\n” <br>

服务器执行命令前需要做若干项检查，具体可通过阅读源码或者查看《redis 设计与实现》中的服务器章节。

#### 调用命令实现
前面的操作，服务器已经将命令参数和命令实现函数都保存在了客户端状态结构中，服务器只需要执行相应的语句即可

{% highlight ruby %}
	void call(redisClient *c, int flags) {
		...
		c->cmd->proc(c);
		...
		/* Log the command into the Slow log if needed, and populate the
	     * per-command statistics that we show in INFO commandstats. */
	    if (flags & REDIS_CALL_SLOWLOG && c->cmd->proc != execCommand) {
	        char *latency_event = (c->cmd->flags & REDIS_CMD_FAST) ?
	                              "fast-command" : "command";
	        latencyAddSampleIfNeeded(latency_event,duration/1000);
	        slowlogPushEntryIfNeeded(c->argv,c->argc,duration);
	    }
	    if (flags & REDIS_CALL_STATS) {
	        c->cmd->microseconds += duration;
	        c->cmd->calls++;
	    }
	
	    /* Propagate the command into the AOF and replication link */
	    if (flags & REDIS_CALL_PROPAGATE) {
	        int flags = REDIS_PROPAGATE_NONE;
	
	        if (c->flags & REDIS_FORCE_REPL) flags |= REDIS_PROPAGATE_REPL;
	        if (c->flags & REDIS_FORCE_AOF) flags |= REDIS_PROPAGATE_AOF;
	        if (dirty)
	            flags |= (REDIS_PROPAGATE_REPL | REDIS_PROPAGATE_AOF);
	        if (flags != REDIS_PROPAGATE_NONE)
	            propagate(c->cmd,c->db->id,c->argv,c->argc,flags);
	    }
		...
	}
{% endhighlight %}

命令执行完之后，还需要做一些其他操作： <br>
如果服务器开启了慢查询日志功能，服务器会检查是否需要为刚执行的命令添加一条慢查询日志； <br>
更新客户端状态属性 milliseconds 和 calls 属性； <br>
如果服务器开启了AOF，那么刚才执行的命令会被写入到AOF缓冲区；<br>
如果其他服务器正在复制当前服务器，那么刚执行的命令会被广播给所有从服务器。


#### 回复命令给客户端
当命令执行完之后，如果是 set、hset类的命令，直接 addReply("+OK\r\n")，回复客户端

如果是 get、hget 类的命令，需要将结果保存在客户端状态结构的输出缓冲区中，然后通过 `sendReplyToClient` 函数返回给客户端。
