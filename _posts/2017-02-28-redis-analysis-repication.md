---
layout: post
title: "redis源码分析 - 复制"
date: 2017-05-26 23:30:00
tags: redis replication
---
redis replication

在 redis 中，用户可以通过执行 SLAVEOF 或者通过设置 slaveof 选项，让一个服务器去复制另一个服务器，我们称呼这为主备复制。

> 查看__[redis主从复制]__：http://blog.wuzhenyu.com.cn/2016/12/15/how-to-build-redis-replication.html <br>
也可以查看我在CSDN 上发的博客: http://blog.csdn.net/honglicu123/article/details/53693395


redis2.8 以上版本的同步，有两种方式的同步，一种为完整重同步(full resychronization)，另一种是部分重同步(partial resychronization)。`PSYNC`具有这两种同步模式。

* 完整重同步用户初次同步复制的情况，通过让主服务器创建并发送RDB文件，以及向从服务器发送保存在缓冲区中的的写命令来进行同步
* 部分重同步，则用于处理断线后重复制的情况。当从服务器与主服务器失去连接后到重新连接主服务器时，如果条件允许，主服务器可以将主从服务器断开期间执行的写命令发送给从服务器，从服务器只需要接收并执行这些写命令，就能将数据库更新至主服务器当前的状态，保持主从服务器数据库状态一致。

## 完整重同步的步骤 (full resynchronization)
完整重同步，与旧版redis 中的 SYNC 命令的复制相同，步骤如下： <br>
1) 从服务器向主服务器发送 SYNC 命令。 <br>
2) 主服务器接收到从服务器发送的SYNC命令之后，执行 BGSAVE 命令，在后台生成一个 RDB 文件，并使用一个缓冲区保存从现在开始执行的所有写命令。 <br>
3) 当主服务器的 BGSAVE 命令执行完毕时，主服务器会将生成的 RDB 文件发送给从服务器，从服务器接收并载入这个 RDB 文件，将自己的数据库状态更新至主服务器执行 BGSAVE 命令时的数据库状态。 <br>
4) 主服务器将缓冲区中的所有写命令发送给从服务器，从服务器接收并执行这些写命令，将自己的数据库状态更新至主服务器当前的数据库状态。

完整重同步，能够很好的完成初次复制和数据同步，但是当从服务器掉线时，如果仍然使用完整重同步，将造成效率低下，占用大量资源，因为这时，只需要同步从服务器掉线期间执行的写命令即可，不需要完整的将整个数据同步一遍。

__缺点__： <br>
1. 主服务器生成 RDB 文件，会占用大量的 CPU、内存和磁盘 I/O 资源 <br>
2. 主服务器发送 RDB 文件，会占用大量的网络资源，这可能会对主服务器相应命令请求造成影响。 <br>
3. 从服务器接收加载 RDB 文件，载入期间，可能会因为阻塞而没办法处理命令请求。

所以，`full resynchronization` 是一个非常耗资源的操作，redis 有必要保证只有在真正需要的时候才执行该操作。

## 部分重同步 (partial resynchronization)
本文以从服务器发送 `slaveof` 命令为例说明 `PSYNC` 的实现。

### 设置主服务器的地址和端口
当从服务器的客户端发送 `slaveof` 命令时，从服务器会将客户端给定的服务器的 IP 地址和端口号保存在服务器状态的 `masterhost` 和 `masterport` 属性里面：

{% highlight ruby %}
	struct redisServer {
		...
		/* Replication (slave) */
	    char *masterauth;               /* AUTH with this password with master */
	    char *masterhost;               /* Hostname of master */
	    int masterport;                 /* Port of master */
	    int repl_timeout;               /* Timeout after N seconds of master idle */
	    redisClient *master;     /* Client that is master for this slave */
	    redisClient *cached_master; /* Cached master to be reused for PSYNC. */
	    int repl_syncio_timeout; /* Timeout for synchronous I/O calls */
	    int repl_state;          /* Replication status if the instance is a slave */
	    off_t repl_transfer_size; /* Size of RDB to read from master during sync. */
	    off_t repl_transfer_read; /* Amount of RDB read from master during sync. */
	    off_t repl_transfer_last_fsync_off; /* Offset when we fsync-ed last time. */
	    int repl_transfer_s;     /* Slave -> Master SYNC socket */
	    int repl_transfer_fd;    /* Slave -> Master SYNC temp file descriptor */
	    char *repl_transfer_tmpfile; /* Slave-> master SYNC temp file name */
	    time_t repl_transfer_lastio; /* Unix time of the latest read, for timeout */
	    int repl_serve_stale_data; /* Serve stale data when link is down? */
	    int repl_slave_ro;          /* Slave is read only? */
	    time_t repl_down_since; /* Unix time at which link with master went down */
	    int repl_disable_tcp_nodelay;   /* Disable TCP_NODELAY after SYNC? */
	    int slave_priority;             /* Reported in INFO and used by Sentinel. */
	    char repl_master_runid[REDIS_RUN_ID_SIZE+1];  /* Master run id for PSYNC. */
	    long long repl_master_initial_offset;         /* Master PSYNC offset. */
	    /* Replication script cache. */
	    dict *repl_scriptcache_dict;        /* SHA1 all slaves are aware of. */
	    list *repl_scriptcache_fifo;        /* First in, first out LRU eviction. */
	    unsigned int repl_scriptcache_size; /* Max number of elements. */
	    /* Synchronous replication. */
	    list *clients_waiting_acks;         /* Clients waiting in WAIT command. */
	    int get_ack_from_slaves;            /* If true we send REPLCONF GETACK. */
		...
	};
{% endhighlight %}

slaveof 是一个异步命令，在完成属性的设置之后，从服务器将向客户端发送 OK，实际的复制工作将从这开始。

### 建立套接字连接
_SLAVEOF_ 命令执行结束后，从服务器将根据命令所设置的 IP 地址和端口，创建连向主服务器的套接字连接。

{% highlight ruby %}
	/* Replication cron function, called 1 time per second. */
	void replicationCron(void) {
		...
		/* Check if we should connect to a MASTER */
	    if (server.repl_state == REDIS_REPL_CONNECT) {
	        redisLog(REDIS_NOTICE,"Connecting to MASTER %s:%d",
	            server.masterhost, server.masterport);
	        if (connectWithMaster() == REDIS_OK) {
	            redisLog(REDIS_NOTICE,"MASTER <-> SLAVE sync started");
	        }
	    }
		...
	}

	int connectWithMaster(void) {
	    int fd;
	
		//create socket connect
	    fd = anetTcpNonBlockBestEffortBindConnect(NULL,
	        server.masterhost,server.masterport,REDIS_BIND_ADDR);
	    if (fd == -1) {
	        redisLog(REDIS_WARNING,"Unable to connect to MASTER: %s",
	            strerror(errno));
	        return REDIS_ERR;
	    }
	
		//create a file event to reponsible for replication between master and slave:
		//比如接收 RDB 文件，接收主服务器传播来的写命令
	    if (aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster,NULL) ==
	            AE_ERR)
	    {
	        close(fd);
	        redisLog(REDIS_WARNING,"Can't create readable event for SYNC");
	        return REDIS_ERR;
	    }
	
	    server.repl_transfer_lastio = server.unixtime;
	    server.repl_transfer_s = fd;
	    server.repl_state = REDIS_REPL_CONNECTING;
	    return REDIS_OK;
	}
{% endhighlight %}

如果从服务器创建的套接字能成功连接到主服务器，那么从服务器将会为这个套接字关联一个文件事件处理器(syncWithMaster)，负责执行后续的复制工作，如接收 RDB 文件，接收服务器传播来的写命令等。

### 发送 PING 命令
从服务器成为主服务器的客户端之后，第一件事就是向主服务器发送 PING 命令。

{% highlight ruby %}
	void replicationCron (void)
	{
		...
		/* If we have attached slaves, PING them from time to time.
	     * So slaves can implement an explicit timeout to masters, and will
	     * be able to detect a link disconnection even if the TCP connection
	     * will not actually go down. */
	    listIter li;
	    listNode *ln;
	    robj *ping_argv[1];
	
	    /* First, send PING according to ping_slave_period. */
	    if ((replication_cron_loops % server.repl_ping_slave_period) == 0) {
	        ping_argv[0] = createStringObject("PING",4);
	        replicationFeedSlaves(server.slaves, server.slaveseldb,
	            ping_argv, 1);
	        decrRefCount(ping_argv[0]);
	    }
	
	    /* Second, send a newline to all the slaves in pre-synchronization
	     * stage, that is, slaves waiting for the master to create the RDB file.
	     * The newline will be ignored by the slave but will refresh the
	     * last-io timer preventing a timeout. In this case we ignore the
	     * ping period and refresh the connection once per second since certain
	     * timeouts are set at a few seconds (example: PSYNC response). */
	    listRewind(server.slaves,&li);
	    while((ln = listNext(&li))) {
	        redisClient *slave = ln->value;
	
	        if (slave->replstate == REDIS_REPL_WAIT_BGSAVE_START ||
	            (slave->replstate == REDIS_REPL_WAIT_BGSAVE_END &&
	             server.rdb_child_type != REDIS_RDB_CHILD_TYPE_SOCKET))
	        {
	            if (write(slave->fd, "\n", 1) == -1) {
	                /* Don't worry, it's just a ping. */
	            }
	        }
	    }
	
	    /* Disconnect timedout slaves. */
	    if (listLength(server.slaves)) {
	        listIter li;
	        listNode *ln;
	
	        listRewind(server.slaves,&li);
	        while((ln = listNext(&li))) {
	            redisClient *slave = ln->value;
	
	            if (slave->replstate != REDIS_REPL_ONLINE) continue;
	            if (slave->flags & REDIS_PRE_PSYNC) continue;
	            if ((server.unixtime - slave->repl_ack_time) > server.repl_timeout)
	            {
	                redisLog(REDIS_WARNING, "Disconnecting timedout slave: %s",
	                    replicationGetSlaveName(slave));
	                freeClient(slave);
	            }
	        }
	    }
		...
	}

	void syncWithMaster(aeEventLoop *el, int fd, void *privdata, int mask) {
		...
		/* Send a PING to check the master is able to reply without errors. */
	    if (server.repl_state == REDIS_REPL_CONNECTING) {
	        redisLog(REDIS_NOTICE,"Non blocking connect for SYNC fired the event.");
	        /* Delete the writable event so that the readable event remains
	         * registered and we can wait for the PONG reply. */
	        aeDeleteFileEvent(server.el,fd,AE_WRITABLE);
	        server.repl_state = REDIS_REPL_RECEIVE_PONG;
	        /* Send the PING, don't check for errors at all, we have the timeout
	         * that will take care about this. */
	        err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"PING",NULL);
	        if (err) goto write_error;
	        return;
	    }
		...
	}
{% endhighlight %}

PING命令的作用: <br>

* 检查套接字的读写状态是否正常
* 检查主服务器能否正常处理命令请求

如果从服务器读取到 "PONG" 回复，说明主从之间网络状态正常，能够进行后续的复制工作，从服务器可以继续执行复制操作的下一个步骤。其他异常情况下，从服务器将断开主服务器的连接，并重新创建连向主服务器的套接字。

{% highlight ruby %}
	/* Receive the PONG command. */
    if (server.repl_state == REDIS_REPL_RECEIVE_PONG) {
        err = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);

        /* We accept only two replies as valid, a positive +PONG reply
         * (we just check for "+") or an authentication error.
         * Note that older versions of Redis replied with "operation not
         * permitted" instead of using a proper error code, so we test
         * both. */
        if (err[0] != '+' &&
            strncmp(err,"-NOAUTH",7) != 0 &&
            strncmp(err,"-ERR operation not permitted",28) != 0)
        {
            redisLog(REDIS_WARNING,"Error reply to PING from master: '%s'",err);
            sdsfree(err);
            goto error;
        } else {
            redisLog(REDIS_NOTICE,
                "Master replied to PING, replication can continue...");
        }
        sdsfree(err);
        server.repl_state = REDIS_REPL_SEND_AUTH;
    }
{% endhighlight %}
### 身份验证
{% highlight ruby %}
	/* AUTH with the master if required. */
	    if (server.repl_state == REDIS_REPL_SEND_AUTH) {
	        if (server.masterauth) {	// "AUTH server.masterauth"
	            err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"AUTH",server.masterauth,NULL);
	            if (err) goto write_error;
	            server.repl_state = REDIS_REPL_RECEIVE_AUTH;
	            return;
	        } else {
	            server.repl_state = REDIS_REPL_SEND_PORT;
	        }
	    }
	
	    /* Receive AUTH reply. */
	    if (server.repl_state == REDIS_REPL_RECEIVE_AUTH) {
	        err = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
	        if (err[0] == '-') {
	            redisLog(REDIS_WARNING,"Unable to AUTH to MASTER: %s",err);
	            sdsfree(err);
	            goto error;
	        }
	        sdsfree(err);
	        server.repl_state = REDIS_REPL_SEND_PORT;
	    }
{% endhighlight %}

从服务器设置了 masterauth 选项，将进行身份验证，否则，不会进行身份验证。但是会出现以下几种情况： <br>

- 主服务器没设置 requirepass 选项，从服务器没有设置 masterauth，主服务能够继续执行从服务器发送的命令请求，复制工作可以继续进行。
- 如果从服务器发送的验证密码与主服务器相同，能够继续进行复制工作；否则，主服务器将返回一个 `invalid password` 的错误
- 主服务器设置了 requirepass 选项，从服务器没有设置 masterauth 选项，那么主服务器将返回一个 `NOAUTH` 的错误；相反，如果主服务器没有设置 requirepass，而从服务器缺设置了 masterauth，那么主服务器将返回一个 `no password is set` 的错误信息。

### 发送端口信息
{% highlight ruby %}
		/* Set the slave port, so that Master's INFO command can list the
	     * slave listening port correctly. */
	    if (server.repl_state == REDIS_REPL_SEND_PORT) {
	        sds port = sdsfromlonglong(server.port);
	        err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"REPLCONF",
	                "listening-port",port, NULL);	// "REPLCONF listening-port 6379"
	        sdsfree(port);
	        if (err) goto write_error;
	        sdsfree(err);
	        server.repl_state = REDIS_REPL_RECEIVE_PORT;
	        return;
	    }
{% endhighlight %}

从服务器发送 `REPLCONF listening-port <port>` ，向主服务器发送从服务器的监听端口号。主服务器接收后，会将端口号记录在从服务器对应的客户端状态结构体中的 `slave_listening_port` 属性中，在客户端执行 `INFO REPLICATION` 命令查看到的 port 参数的值就是这个属性的值。

### 同步
{% highlight ruby %}
		/* Try a partial resynchonization. If we don't have a cached master
	     * slaveTryPartialResynchronization() will at least try to use PSYNC
	     * to start a full resynchronization so that we get the master run id
	     * and the global offset, to try a partial resync at the next
	     * reconnection attempt. */
	    if (server.repl_state == REDIS_REPL_SEND_PSYNC) {
	        if (slaveTryPartialResynchronization(fd,0) == PSYNC_WRITE_ERROR) {
	            err = sdsnew("Write error sending the PSYNC command.");
	            goto write_error;
	        }
	        server.repl_state = REDIS_REPL_RECEIVE_PSYNC;
	        return;
	    }
	
	    /* If reached this point, we should be in REDIS_REPL_RECEIVE_PSYNC. */
	    if (server.repl_state != REDIS_REPL_RECEIVE_PSYNC) {
	        redisLog(REDIS_WARNING,"syncWithMaster(): state machine error, "
	                             "state should be RECEIVE_PSYNC but is %d",
	                             server.repl_state);
	        goto error;
	    }
	
	    psync_result = slaveTryPartialResynchronization(fd,1);
	    if (psync_result == PSYNC_WAIT_REPLY) return; /* Try again later... */
	
	    /* Note: if PSYNC does not return WAIT_REPLY, it will take care of
	     * uninstalling the read handler from the file descriptor. */
	
	    if (psync_result == PSYNC_CONTINUE) {
	        redisLog(REDIS_NOTICE, "MASTER <-> SLAVE sync: Master accepted a Partial Resynchronization.");
	        return;
	    }
	
	    /* PSYNC failed or is not supported: we want our slaves to resync with us
	     * as well, if we have any (chained replication case). The mater may
	     * transfer us an entirely different data set and we have no way to
	     * incrementally feed our slaves after that. */
	    disconnectSlaves(); /* Force our slaves to resync with us as well. */
	    freeReplicationBacklog(); /* Don't allow our chained slaves to PSYNC. */
	
	    /* Fall back to SYNC if needed. Otherwise psync_result == PSYNC_FULLRESYNC
	     * and the server.repl_master_runid and repl_master_initial_offset are
	     * already populated. */
	    if (psync_result == PSYNC_NOT_SUPPORTED) {
	        redisLog(REDIS_NOTICE,"Retrying with SYNC...");
	        if (syncWrite(fd,"SYNC\r\n",6,server.repl_syncio_timeout*1000) == -1) {
	            redisLog(REDIS_WARNING,"I/O error writing to MASTER: %s",
	                strerror(errno));
	            goto error;
	        }
	    }
	
	    /* Prepare a suitable temp file for bulk transfer */
	    while(maxtries--) {
	        snprintf(tmpfile,256,
	            "temp-%d.%ld.rdb",(int)server.unixtime,(long int)getpid());
	        dfd = open(tmpfile,O_CREAT|O_WRONLY|O_EXCL,0644);
	        if (dfd != -1) break;
	        sleep(1);
	    }
	    if (dfd == -1) {
	        redisLog(REDIS_WARNING,"Opening the temp file needed for MASTER <-> SLAVE synchronization: %s",strerror(errno));
	        goto error;
	    }
	
	    /* Setup the non blocking download of the bulk file. */
	    if (aeCreateFileEvent(server.el,fd, AE_READABLE,readSyncBulkPayload,NULL)
	            == AE_ERR)
	    {
	        redisLog(REDIS_WARNING,
	            "Can't create readable event for SYNC: %s (fd=%d)",
	            strerror(errno),fd);
	        goto error;
	    }
{% endhighlight %}

按照上文代码中的注释，如果是初次复制，`we don't have a cached master`，采用的是 `full resynchronization`，获取 `master run id and the global offset`。如果是断线重连复制，使用的部分重复制 `partial resynchronization`。使用 `full resynchronization` 时，接收主服务器发送的 RDB 文件。

{% highlight ruby %}
	#define PSYNC_WRITE_ERROR 0
	#define PSYNC_WAIT_REPLY 1
	#define PSYNC_CONTINUE 2
	#define PSYNC_FULLRESYNC 3
	#define PSYNC_NOT_SUPPORTED 4
	int slaveTryPartialResynchronization(int fd, int read_reply) {
	    char *psync_runid;
	    char psync_offset[32];
	    sds reply;
	
	    /* Writing half */
	    if (!read_reply) {
	        /* Initially set repl_master_initial_offset to -1 to mark the current
	         * master run_id and offset as not valid. Later if we'll be able to do
	         * a FULL resync using the PSYNC command we'll set the offset at the
	         * right value, so that this information will be propagated to the
	         * client structure representing the master into server.master. */
	        server.repl_master_initial_offset = -1;
	
	        if (server.cached_master) {
	            psync_runid = server.cached_master->replrunid;
	            snprintf(psync_offset,sizeof(psync_offset),"%lld", server.cached_master->reploff+1);
	            redisLog(REDIS_NOTICE,"Trying a partial resynchronization (request %s:%s).", psync_runid, psync_offset);
	        } else {
	            redisLog(REDIS_NOTICE,"Partial resynchronization not possible (no cached master)");
	            psync_runid = "?";
	            memcpy(psync_offset,"-1",3);
	        }
	
	        /* Issue the PSYNC command */
			/* PSYNC ? -1 */
	        reply = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"PSYNC",psync_runid,psync_offset,NULL);
	        if (reply != NULL) {
	            redisLog(REDIS_WARNING,"Unable to send PSYNC to master: %s",reply);
	            sdsfree(reply);
	            aeDeleteFileEvent(server.el,fd,AE_READABLE);
	            return PSYNC_WRITE_ERROR;
	        }
	        return PSYNC_WAIT_REPLY;
	    }
	
	    /* Reading half */
	    reply = sendSynchronousCommand(SYNC_CMD_READ,fd,NULL);
	    if (sdslen(reply) == 0) {
	        /* The master may send empty newlines after it receives PSYNC
	         * and before to reply, just to keep the connection alive. */
	        sdsfree(reply);
	        return PSYNC_WAIT_REPLY;
	    }
	
	    aeDeleteFileEvent(server.el,fd,AE_READABLE);
	
	    if (!strncmp(reply,"+FULLRESYNC",11)) {
	        char *runid = NULL, *offset = NULL;
	
	        /* FULL RESYNC, parse the reply in order to extract the run id
	         * and the replication offset. */
	        runid = strchr(reply,' ');
	        if (runid) {
	            runid++;
	            offset = strchr(runid,' ');
	            if (offset) offset++;
	        }
	        if (!runid || !offset || (offset-runid-1) != REDIS_RUN_ID_SIZE) {
	            redisLog(REDIS_WARNING,
	                "Master replied with wrong +FULLRESYNC syntax.");
	            /* This is an unexpected condition, actually the +FULLRESYNC
	             * reply means that the master supports PSYNC, but the reply
	             * format seems wrong. To stay safe we blank the master
	             * runid to make sure next PSYNCs will fail. */
	            memset(server.repl_master_runid,0,REDIS_RUN_ID_SIZE+1);
	        } else {
	            memcpy(server.repl_master_runid, runid, offset-runid-1);
	            server.repl_master_runid[REDIS_RUN_ID_SIZE] = '\0';
	            server.repl_master_initial_offset = strtoll(offset,NULL,10);
	            redisLog(REDIS_NOTICE,"Full resync from master: %s:%lld",
	                server.repl_master_runid,
	                server.repl_master_initial_offset);
	        }
	        /* We are going to full resync, discard the cached master structure. */
	        replicationDiscardCachedMaster();
	        sdsfree(reply);
	        return PSYNC_FULLRESYNC;
	    }
	
	    if (!strncmp(reply,"+CONTINUE",9)) {
	        /* Partial resync was accepted, set the replication state accordingly */
	        redisLog(REDIS_NOTICE,
	            "Successful partial resynchronization with master.");
	        sdsfree(reply);
	        replicationResurrectCachedMaster(fd);
	        return PSYNC_CONTINUE;
	    }
	
	    /* If we reach this point we received either an error since the master does
	     * not understand PSYNC, or an unexpected reply from the master.
	     * Return PSYNC_NOT_SUPPORTED to the caller in both cases. */
	
	    if (strncmp(reply,"-ERR",4)) {
	        /* If it's not an error, log the unexpected event. */
	        redisLog(REDIS_WARNING,
	            "Unexpected reply to PSYNC from master: %s", reply);
	    } else {
	        redisLog(REDIS_NOTICE,
	            "Master does not support PSYNC or is in "
	            "error state (reply: %s)", reply);
	    }
	    sdsfree(reply);
	    replicationDiscardCachedMaster();
	    return PSYNC_NOT_SUPPORTED;
	}
{% endhighlight %}

slaveTryPartialResynchronization 函数描述了主服务器接收到 `PSYNC` 命令时，返回给从服务器的几种情况。 如果从服务器与主服务器是初次复制，或者之前执行过 `slaveof no one` 命令，那么从服务器将向主服务器发送 `PSYNC ? -1` 命令，请求进行__完整重复制__；否则，从服务器向主服务器发送 `PSYNC <runid> <offset>` 命令，请求进行__部分重同步__。

- 如果主服务器返回 `+FULLRESYNC <runid> <offset>` 回复，表示主从将执行完整重同步。 runid 为主服务的 runid，从服务器保存这个值，用于下次发送 PSYNC 命令时使用，offset 是主服务器当前的复制偏移量，从服务器会将这个值作为自己的初始化偏移值。
- 如果主服务器返回 `+CONTINUE` ，进行部分重同步
- 返回 `-ERR`，表示主服务器版本低于 2.8，不能识别 `PSYNC` 命令，使用 `SYNC` 进行完整重同步操作。
### 命令传播
当完成同步之后，主从服务器就会进入命令传播阶段。这时，主服务器只要一直将自己执行的写命令发送给从服务器，从服务器只需要一直接收和执行主服务器发送过来的写命令，就可以保证主从服务器数据库状态一致了。

{% highlight ruby %}
	void syncCommand (redisClient* c)
	{
		...
		/* Try a partial resynchronization if this is a PSYNC command.
	     * If it fails, we continue with usual full resynchronization, however
	     * when this happens masterTryPartialResynchronization() already
	     * replied with:
	     *
	     * +FULLRESYNC <runid> <offset>
	     *
	     * So the slave knows the new runid and offset to try a PSYNC later
	     * if the connection with the master is lost. */
	    if (!strcasecmp(c->argv[0]->ptr,"psync")) {
	        if (masterTryPartialResynchronization(c) == REDIS_OK) {
	            server.stat_sync_partial_ok++;
	            return; /* No full resync needed, return. */
	        } else {
	            char *master_runid = c->argv[1]->ptr;
	
	            /* Increment stats for failed PSYNCs, but only if the
	             * runid is not "?", as this is used by slaves to force a full
	             * resync on purpose when they are not albe to partially
	             * resync. */
	            if (master_runid[0] != '?') server.stat_sync_partial_err++;
	        }
	    } else {
	        /* If a slave uses SYNC, we are dealing with an old implementation
	         * of the replication protocol (like redis-cli --slave). Flag the client
	         * so that we don't expect to receive REPLCONF ACK feedbacks. */
	        c->flags |= REDIS_PRE_PSYNC;
	    }
		...
	}
{% endhighlight %}

复制积压缓冲区，就是一个循环数组，可以看成是一个队列，通过先进先出的方式，如果数组满了，会将最开始的那部分覆盖。

{% highlight ruby %}
	/* Feed the slave 'c' with the replication backlog starting from the
	 * specified 'offset' up to the end of the backlog. */
	long long addReplyReplicationBacklog(redisClient *c, long long offset) {
	    long long j, skip, len;
	
	    redisLog(REDIS_DEBUG, "[PSYNC] Slave request offset: %lld", offset);
	
	    if (server.repl_backlog_histlen == 0) {
	        redisLog(REDIS_DEBUG, "[PSYNC] Backlog history len is zero");
	        return 0;
	    }
	
	    redisLog(REDIS_DEBUG, "[PSYNC] Backlog size: %lld",
	             server.repl_backlog_size);
	    redisLog(REDIS_DEBUG, "[PSYNC] First byte: %lld",
	             server.repl_backlog_off);
	    redisLog(REDIS_DEBUG, "[PSYNC] History len: %lld",
	             server.repl_backlog_histlen);
	    redisLog(REDIS_DEBUG, "[PSYNC] Current index: %lld",
	             server.repl_backlog_idx);
	
	    /* Compute the amount of bytes we need to discard. */
	    skip = offset - server.repl_backlog_off;
	    redisLog(REDIS_DEBUG, "[PSYNC] Skipping: %lld", skip);
	
	    /* Point j to the oldest byte, that is actaully our
	     * server.repl_backlog_off byte. */
	    j = (server.repl_backlog_idx +
	        (server.repl_backlog_size-server.repl_backlog_histlen)) %
	        server.repl_backlog_size;
	    redisLog(REDIS_DEBUG, "[PSYNC] Index of first byte: %lld", j);
	
	    /* Discard the amount of data to seek to the specified 'offset'. */
	    j = (j + skip) % server.repl_backlog_size;
	
	    /* Feed slave with data. Since it is a circular buffer we have to
	     * split the reply in two parts if we are cross-boundary. */
	    len = server.repl_backlog_histlen - skip;
	    redisLog(REDIS_DEBUG, "[PSYNC] Reply total length: %lld", len);
	    while(len) {
	        long long thislen =
	            ((server.repl_backlog_size - j) < len) ?
	            (server.repl_backlog_size - j) : len;
	
	        redisLog(REDIS_DEBUG, "[PSYNC] addReply() length: %lld", thislen);
	        addReplySds(c,sdsnewlen(server.repl_backlog + j, thislen));
	        len -= thislen;
	        j = 0;
	    }
	    return server.repl_backlog_histlen - skip;
	}
{% endhighlight %}
## 心跳检测
在命令传播阶段，从服务器会默认以每秒一次的频率，向主服务器发送命令：

	REPLCONF ACK <replication_offset>
	
`replication_offset` 是从服务器当前的复制偏移量。发送该命令的作用：

* 检测主从服务器的网络连接状态
* 辅助实现 min-slaves
* 检测命令丢失

replication.c 中的 `replicationCron` 函数每秒执行一次，

	void replicationCron (void)
	{
		...
	    /* Send ACK to master from time to time.
	     * Note that we do not send periodic acks to masters that don't
	     * support PSYNC and replication offsets. */
	    if (server.masterhost && server.master &&
	        !(server.master->flags & REDIS_PRE_PSYNC))
	        replicationSendAck();
	
		...
	}
	
从中可知， redis 从服务器会每秒向主服务器发送一次 ACK

{% highlight ruby %}
	/* Send a REPLCONF ACK command to the master to inform it about the current
	 * processed offset. If we are not connected with a master, the command has
	 * no effects. */
	void replicationSendAck(void) {
	    redisClient *c = server.master;
	
	    if (c != NULL) {
	        c->flags |= REDIS_MASTER_FORCE_REPLY;
	        addReplyMultiBulkLen(c,3);
	        addReplyBulkCString(c,"REPLCONF");
	        addReplyBulkCString(c,"ACK");
	        addReplyBulkLongLong(c,c->reploff);
	        c->flags &= ~REDIS_MASTER_FORCE_REPLY;
	    }
	}
{% endhighlight %}

`reploff` 是从服务器的复制偏移量
### 检测主从服务器的网络连接状态
如果主服务器超过1秒钟没有接收到从服务器发送的 `REPLCONF ACK` 命令，那么主服务器就认为主从服务器之间的网络连接出现了问题。

通过向主服务器发送 `INFO REPLICATION` ，在列出的参数说明的 lag 一栏中，就表示从服务器最后一次向主服务器发送 `REPLCONF ACK` 命令距离现在过了多少秒。

### 辅助实现 min-slaves
在 redis 配置文件中，

	min-slaves-to-write 3
	min-slaves-max-lag 10
	
这两个参数，`require at least 3 slaves with a lag <= 10 seconds`，也就是说，当从服务器的数量少于三个或者三个从服务器的延迟 (lag) 都大于等于 10 秒时，主服务器将拒绝执行写命令。

在 redis.c 的 `processCommand` 函数中实现

{% highlight ruby %}
	int processCommand (redisClient *c)
	{
		...
	    /* Don't accept write commands if there are not enough good slaves and
	     * user configured the min-slaves-to-write option. */
	    if (server.masterhost == NULL &&
	        server.repl_min_slaves_to_write &&
	        server.repl_min_slaves_max_lag &&
	        c->cmd->flags & REDIS_CMD_WRITE &&
	        server.repl_good_slaves_count < server.repl_min_slaves_to_write)
	    {
	        flagTransaction(c);
	        addReply(c, shared.noreplicaserr);	//-NOREPLICAS Not enough good slaves to write.\r\n
	        return REDIS_OK;
	    }
		...
	}
{% endhighlight %}

如果不满足条件，主服务器将返回 `-NOREPLICAS Not enough good slaves to write.`

在 replication.c 的 `refreshGoodSlavesCount(void)` 函数中，会对 `repl_good_slaves_count` 这个属性进行更新。

{% highlight ruby %}
	/* This function counts the number of slaves with lag <= min-slaves-max-lag.
	 * If the option is active, the server will prevent writes if there are not
	 * enough connected slaves with the specified lag (or less). */
	void refreshGoodSlavesCount(void) {
	    listIter li;
	    listNode *ln;
	    int good = 0;
	
	    if (!server.repl_min_slaves_to_write ||
	        !server.repl_min_slaves_max_lag) return;
	
	    listRewind(server.slaves,&li);
	    while((ln = listNext(&li))) {
	        redisClient *slave = ln->value;
	        time_t lag = server.unixtime - slave->repl_ack_time;
	
	        if (slave->replstate == REDIS_REPL_ONLINE &&
	            lag <= server.repl_min_slaves_max_lag) good++;
	    }
	    server.repl_good_slaves_count = good;
	}
{% endhighlight %}
