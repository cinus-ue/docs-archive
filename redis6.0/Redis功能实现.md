# Redis功能实现

## Redis Server启动过程
代码：server.c
```c
int main(int argc, char **argv) {
    ...
	// 初始化配置
	initServerConfig();
    ...
    // 监听端口，初始化数据
    initServer()
    ...
    // 运行事件处理器
    aeMain()
    // 服务器关闭，停止事件循环
    aeDeleteEventLoop(server.el);
    return 0;
}
```

## 事件循环
aeProcessEvents 函数中最重要的两个动作分别是对 aeApiPoll 的调用和对 processTimeEvents 的调用：  
1.aeApiPoll 获取所有可以不阻塞处理的文件事件  
2.processTimeEvents 执行所有可运行的时间事件  

```c
// Redis的事件处理主循环由aeMain函数进行
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
         // 开始处理事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}

int aeProcessEvents(aeEventLoop *eventLoop, int flags){
    ...
    /* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
    numevents = aeApiPoll(eventLoop, tvp);
    ...
    for (j = 0; j < numevents; j++) {
        ...
        /* Fire the readable event if the call sequence is not inverted. */
        if (!invert && fe->mask & mask & AE_READABLE) {
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
            fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
        }

        /* Fire the writable event. */
        if (fe->mask & mask & AE_WRITABLE) {
            if (!fired || fe->wfileProc != fe->rfileProc) {
                fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
        }
        ...
    }
    // 处理时间事件
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);
    return processed; /* return the number of processed file/time events */
}
```

## 协议层
初始化server时，创建aeCreateFileEvent（aeFileEvent的一种），当accept（可读事件的一种）就绪时，触发aeCreateFileEvent->rfileProc方法，也就是acceptTcpHandler
```c
    //创建监听端口
    /* Open the TCP listening socket for the user commands. */
    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)
        exit(1);
    if (server.tls_port != 0 &&
        listenToPort(server.tls_port,server.tlsfd,&server.tlsfd_count) == C_ERR)
        exit(1);

    // 创建好的监听描述符保存在描述符数组server.ipfd中，最后创建的监听描述符的总数为server.ipfd_count。
    // 注册建连事件
    /* Create an event handler for accepting new connections in TCP and Unix
     * domain sockets. */
    for (j = 0; j < server.ipfd_count; j++) {
        // 针对每个监听描述符，调用aeCreateFileEvent，注册其上的可读事件，回调函数为acceptTcpHandler
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }

int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];
    //添加监听的事件
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

Redis服务器收到客户端的TCP连接后，就会调用acceptTcpHandler函数进行处理  

acceptTcpHandler ==> createClient ==> aeCreateFileEvent ==> readQueryFromClient
```c
#define MAX_ACCEPTS_PER_CALL 1000
// 该函数每次最多处理MAX_ACCEPTS_PER_CALL(1000)个连接，如果还有其他连接，则等到下次调用acceptTcpHandler时再处理，这样做的原因是为了保证该函数的执行时间不会过长，以免影响后续事件的处理
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[NET_IP_STR_LEN];
    UNUSED(el);
    UNUSED(mask);
    UNUSED(privdata);

    while(max--) {
        // 针对每个连接，调用anetTcpAccept函数进行accept，并将客户端地址记录到cip以及cport中
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                serverLog(LL_WARNING,
                    "Accepting client connection: %s", server.neterr);
            return;
        }
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
        // 建立连接后的socket描述符为cfd，根据该值调用acceptCommonHandler，该函数中，调用createClient创建一个redisClient结构
        acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip);
    }
}

static void acceptCommonHandler(connection *conn, int flags, char *ip) {
    ...
    // 创建客户端
    /* Create connection and client */
    if ((c = createClient(conn)) == NULL) {
        serverLog(LL_WARNING,
            "Error registering fd event for the new client: %s (conn: %s)",
            connGetLastError(conn),
            connGetInfo(conn, conninfo, sizeof(conninfo)));
        connClose(conn); /* May be already closed, just ignore errors */
        return;
    }
}

client *createClient(connection *conn) {
    client *c = zmalloc(sizeof(client));

    /* passing NULL as conn it is possible to create a non connected client.
     * This is useful since all the commands needs to be executed
     * in the context of a client. When commands are executed in other
     * contexts (for instance a Lua script) we need a non connected client. */
    if (conn) {
        connNonBlock(conn);
        connEnableTcpNoDelay(conn);
        if (server.tcpkeepalive)
            connKeepAlive(conn,server.tcpkeepalive);
        // 注册readQueryFromClient函数
        connSetReadHandler(conn, readQueryFromClient);  
        connSetPrivateData(conn, c);
    }
    ...
    return c;
}

/* Register a read handler, to be called when the connection is readable.
 * If NULL, the existing handler is removed.
 */
static inline int connSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
    return conn->type->set_read_handler(conn, func);
}
```

## IO多线程
Redis的IO多线程只是用来处理网络数据的读写和协议解析，执行命令仍然是单线程。  

加入多线程 IO 之后，整体的读流程如下:  
1.主线程负责接收建连请求，读事件到来(收到请求)则放到一个全局等待读处理队列  
2.主线程处理完读事件之后，通过RR(Round Robin)将这些连接分配给这些IO线程，然后主线程忙等待(spinlock 的效果)状态  
3.IO 线程将请求数据读取并解析完成(这里只是读数据和解析并不执行)  
4.主线程执行所有命令并清空整个请求等待读处理队列(执行部分串行）   

```c
/* Initialize the data structures needed for threaded I/O. */
void initThreadedIO(void) {
    server.io_threads_active = 0; /* We start with threads not active. */
    // 控制线程个数
    ...
    /* Spawn and initialize the I/O threads. */
    for (int i = 0; i < server.io_threads_num; i++) {
        /* Things we do for all the threads including the main thread. */
        io_threads_list[i] = listCreate();
        if (i == 0) continue; /* Thread 0 is the main thread. */

        /* Things we do only for the additional threads. */
        pthread_t tid;
        pthread_mutex_init(&io_threads_mutex[i],NULL);
        setIOPendingCount(i, 0);
        pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
        // 创建线程
        if (pthread_create(&tid,NULL,IOThreadMain,(void*)(long)i) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize IO thread.");
            exit(1);
        }
        io_threads[i] = tid;
    }
}

IO 处理线程入口
void *IOThreadMain(void *myid) {
    ...
    while(1) {
        ...
        /* Process: note that the main thread will never touch our list
         * before we drop the pending count to 0. */
        listIter li;
        listNode *ln;
        // 遍历线程id获取线程对应的待处理连接列表
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            // 通过io_threads_op，判断读写类型
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                // 读取客户端查询请求
                readQueryFromClient(c->conn);  
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        ...
    }
}

int handleClientsWithPendingReadsUsingThreads(void) {
    ...
    // 一直等待直到所有的连接请求都被 IO 线程处理完
    /* Wait for all the other threads to end their work. */
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }
    if (tio_debug) printf("I/O READ All threads finshed\n");
    /* Run the list of clients again to process the new buffers. */
    while(listLength(server.clients_pending_read)) {
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_READ;
        listDelNode(server.clients_pending_read,ln);

        if (c->flags & CLIENT_PENDING_COMMAND) {
            c->flags &= ~CLIENT_PENDING_COMMAND;
            if (processCommandAndResetClient(c) == C_ERR) {
                /* If the client is no longer valid, we avoid
                 * processing the client later. So we just go
                 * to the next. */
                continue;
            }
        }
        processInputBuffer(c);
    }

    /* Update processed count on server */
    server.stat_io_reads_processed += processed;
}
```

## 接收请求
从redis客户端发送过来的命令，都会在readQueryFromClient函数中被读取。当客户端和服务器的连接套接字变的可读的时候，就会触发redis的文件事件。
在readQueryFromClient函数中，需要完成了2件事情:  
1.将命令的内容读取到redis客户端数据结构中的查询缓冲区（querybuf），该缓存会根据接收到的数据长度动态扩容。  
2.调用processInputBuffer函数，根据协议格式，得出命令的参数等信息。  

在得到一条完整的命令请求数据后，就调用processCommandAndResetClient、processCommand函数处理执行相应的命令。
```c
#define PROTO_IOBUF_LEN (1024*16)  /* Generic I/O buffer size */
void readQueryFromClient(connection *conn) {
    /* Check if we want to read from the client later when exiting from
     * the event loop. This is the case if threaded I/O is enabled. */
    // 启用多线程，直接返回
    // Return 1 if we want to handle the client read later using threaded I/O.
    // 函数postponeClientRead()将任务放入处理队列，而根据上面IOThreadMain()和 handleClientsWithPendingReadsUsingThreads()的任务处理逻辑进行处理
    if (postponeClientRead(c)) return; 
    // 多线程未启用，则执行后面的逻辑
    // 读入长度
    readlen = PROTO_IOBUF_LEN;
    ...
    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    // 读入内容到查询缓存
    nread = connRead(c->conn, c->querybuf+qblen, readlen);
    ...
    /* There is more data in the client input buffer, continue parsing it
     * in case to check if there is a full command to execute. */
     processInputBuffer(c);
}

void processInputBuffer(client *c) {
    // 判断请求类型，两种类型的区别可以查询Redis的通讯协议
    /* Determine request type when unknown. */
    if (!c->reqtype) {
        if (c->querybuf[c->qb_pos] == '*') {
            // 多条查询
            c->reqtype = PROTO_REQ_MULTIBULK;
        } else {
            // 内联查询
            c->reqtype = PROTO_REQ_INLINE;
        }
    }
    // 根据请求的类型，调用不同的处理函数
    // 将缓冲区中的内容转换成命令，以及命令参数
    if (c->reqtype == PROTO_REQ_INLINE) {
        if (processInlineBuffer(c) != C_OK) break;
        ...
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != C_OK) break;
        } else {
            serverPanic("Unknown request type");
        }

        /* Multibulk processing could see a <= 0 length. */
        if (c->argc == 0) {
            resetClient(c);
        } else {
            ...
            /* We are finally ready to execute the command. */
            // 执行命令，并重置客户端
            if (processCommandAndResetClient(c) == C_ERR) {
                /* If the client is no longer valid, we avoid exiting this
                * loop and trimming the client buffer later. So we return
                * ASAP in that case. */
                return;
            }
        }
    }
}
```

processMultibulkBuffer主要完成的工作是将 c->querybuf 中的协议内容转换成 c->argv 中的参数对象。 比如 *3\r\n$3\r\nSET\r\n$3\r\nMSG\r\n$5\r\nHELLO\r\n将被转换为：
```
 argv[0] = SET
 argv[1] = MSG
 argv[2] = HELLO
```
```c
int processMultibulkBuffer(client *c) {
    char *newline = NULL;
    int ok;
    long long ll;
    // 读入命令的参数个数
	// 比如 *3\r\n$3\r\nSET\r\n... 将令 c->multibulklen = 3
    if (c->multibulklen == 0) {
        /* The client should have been reset */
        serverAssertWithInfo(c,NULL,c->argc == 0);

        // 检查缓冲区是否包含 "\r\n"
        /* Multi bulk length cannot be read without a \r\n */
        newline = strchr(c->querybuf+c->qb_pos,'\r');
        if (newline == NULL) {
            ...
            return C_ERR;
        }

        /* Buffer should also contain \n */
        if (newline-(c->querybuf+c->qb_pos) > (ssize_t)(sdslen(c->querybuf)-c->qb_pos-2))
            return C_ERR;

        /* We know for sure there is a whole line since newline != NULL,
         * so go ahead and find out the multi bulk length. */
        serverAssertWithInfo(c,NULL,c->querybuf[c->qb_pos] == '*');
        // 协议的第一个字符必须是 '*'
		// 将参数个数，也即是 * 之后， \r\n 之前的数字取出并保存到 ll 中
		// 比如对于 *3\r\n ，那么 ll 将等于 3
        ok = string2ll(c->querybuf+1+c->qb_pos,newline-(c->querybuf+1+c->qb_pos),&ll);
        // 参数的数量超出限制
        if (!ok || ll > 1024*1024) {
            addReplyError(c,"Protocol error: invalid multibulk length");
            setProtocolError("invalid mbulk count",c);
            return C_ERR;
        }

        c->qb_pos = (newline-c->querybuf)+2;

        if (ll <= 0) return C_OK;
        // 设置参数数量
		// 根据参数数量，为各个参数对象分配空间
        c->multibulklen = ll;

        /* Setup argv array on client structure */
        if (c->argv) zfree(c->argv);
        c->argv = zmalloc(sizeof(robj*)*c->multibulklen);
    }

    // 从 c->querybuf 中读入参数，并创建各个参数对象到 c->argv
    while(c->multibulklen) {
        /* Read bulk length if unknown */
        if (c->bulklen == -1) {
            // 确保 "\r\n" 存在
            newline = strchr(c->querybuf+c->qb_pos,'\r');
            if (newline == NULL) {
                ...
                break;
            }

            /* Buffer should also contain \n */
            if (newline-(c->querybuf+c->qb_pos) > (ssize_t)(sdslen(c->querybuf)-c->qb_pos-2))
                break;

            // 确保协议符合参数格式，检查其中的 $...
            if (c->querybuf[c->qb_pos] != '$') {
                ...
                return C_ERR;
            }
            // 读取长度
			// 比如 $3\r\nSET\r\n 将会让 ll 的值设置 3
            ok = string2ll(c->querybuf+c->qb_pos+1,newline-(c->querybuf+c->qb_pos+1),&ll);
            if (!ok || ll < 0 ||
                (!(c->flags & CLIENT_MASTER) && ll > server.proto_max_bulk_len)) {
                ...
                return C_ERR;
            }

            c->qb_pos = newline-c->querybuf+2;
            if (ll >= PROTO_MBULK_BIG_ARG) {
                if (sdslen(c->querybuf)-c->qb_pos <= (size_t)ll+2) {
                    sdsrange(c->querybuf,c->qb_pos,-1);
                    c->qb_pos = 0;
                    /* Hint the sds library about the amount of bytes this string is
                     * going to contain. */
                    c->querybuf = sdsMakeRoomFor(c->querybuf,ll+2);
                }
            }
            // 参数的长度
            c->bulklen = ll;
        }

        // 读入参数
		// 确保内容符合协议格式
		// 为参数创建字符串对象  
        /* Read bulk argument */
        if (sdslen(c->querybuf)-c->qb_pos < (size_t)(c->bulklen+2)) {
            /* Not enough data (+2 == trailing \r\n) */
            break;
        } else {
            /* Optimization: if the buffer contains JUST our bulk element
             * instead of creating a new object by *copying* the sds we
             * just use the current sds string. */
            if (c->qb_pos == 0 &&
                c->bulklen >= PROTO_MBULK_BIG_ARG &&
                sdslen(c->querybuf) == (size_t)(c->bulklen+2))
            {
                c->argv[c->argc++] = createObject(OBJ_STRING,c->querybuf);
                sdsIncrLen(c->querybuf,-2); /* remove CRLF */
                /* Assume that if we saw a fat argument we'll see another one
                 * likely... */
                c->querybuf = sdsnewlen(SDS_NOINIT,c->bulklen+2);
                sdsclear(c->querybuf);
            } else {
                c->argv[c->argc++] =
                    createStringObject(c->querybuf+c->qb_pos,c->bulklen);
                c->qb_pos += c->bulklen+2;
            }
            // 清空参数长度
		    // 减少还需读入的参数个数
            c->bulklen = -1;
            c->multibulklen--;
        }
    }

	// 如果本条命令的所有参数都已读取完，那么返回
    /* We're done when c->multibulk == 0 */
    if (c->multibulklen == 0) return C_OK;

	// 如果还有参数未读取完，那么说明内容有错，不符合协议
    /* Still not ready to process the command */
    return C_ERR;
}
```

## 初始化commands
```c
/* Command table -- we initialize it here as it is part of the
 * initial configuration, since command names may be changed via
 * redis.conf using the rename-command directive. */
server.commands = dictCreate(&commandTableDictType,NULL);

struct redisCommand redisCommandTable[] = {
    {"module",moduleCommand,-2,
     "admin no-script",
     0,NULL,0,0,0,0,0,0},

    {"get",getCommand,2,
     "read-only fast @string",
     0,NULL,1,1,1,0,0,0},
    ...

/* Populates the Redis Command Table starting from the hard coded list
 * we have on top of server.c file. */
void populateCommandTable(void) {
    int j;
    int numcommands = sizeof(redisCommandTable)/sizeof(struct redisCommand);

    for (j = 0; j < numcommands; j++) {
        struct redisCommand *c = redisCommandTable+j;
        int retval1, retval2;

        /* Translate the command string flags description into an actual
         * set of flags. */
        if (populateCommandTableParseFlags(c,c->sflags) == C_ERR)
            serverPanic("Unsupported command flag");

        c->id = ACLGetCommandID(c->name); /* Assign the ID used for ACL. */
        retval1 = dictAdd(server.commands, sdsnew(c->name), c);
        /* Populate an additional dictionary that will be unaffected
         * by rename-command statements in redis.conf. */
        retval2 = dictAdd(server.orig_commands, sdsnew(c->name), c);
        serverAssert(retval1 == DICT_OK && retval2 == DICT_OK);
    }
}
```

## 执行命令
```c
int processCommand(client *c) {
    // 特别处理 quit 命令
    // 查找命令，并进行命令合法性检查，以及命令参数个数检查
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    // 没找到指定的命令 或 参数个数错误 直接返回
    // 检查认证信息
    // 如果开启了集群模式，那么在这里进行转向操作。
    // 如果设置了最大内存，那么检查内存是否超过限制，并做相应的操作
    /* Don't accept write commands if there are problems persisting on disk
     * and if this is a master instance. */
    /* Don't accept write commands if there are not enough good slaves and
     * user configured the min-slaves-to-write option. */
    // 如果这个服务器是一个只读 slave 的话，那么拒绝执行写命令
    // 在订阅于发布模式的上下文中，只能执行订阅和退订相关的命令
    /* Only allow commands with flag "t", such as INFO, SLAVEOF and so on,
     * when slave-serve-stale-data is no and we are a slave with a broken
     * link with master. */
    /* Loading DB? Return an error if the command has not the
     * CMD_LOADING flag. */
    /* Lua script too slow? Only allow a limited number of commands. */
    // Lua 脚本超时，只允许执行限定的操作，比如 SHUTDOWN 和 SCRIPT KILL
    ...
   /* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        // 在事务上下文中除 EXEC 、 DISCARD 、 MULTI 和 WATCH 命令之外
        // 其他所有命令都会被入队到事务队列中
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        // 执行命令
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            // 处理那些解除了阻塞的键
            handleClientsBlockedOnKeys();
    }
    return C_OK;
}
```

## 查找命令
```c
//  查找命令
struct redisCommand *lookupCommand(sds name) {
    return dictFetchValue(server.commands, name);
}
```

```c
void call(client *c, int flags) {
    // start 记录命令开始执行的时间
	// 记录命令开始执行前的 FLAG
	// 如果可以的话，将命令发送到 MONITOR
    ...
    /* Call the command. */
    dirty = server.dirty;
    updateCachedTime(0);
    start = server.ustime; // 命令开始执行的时间
    c->cmd->proc(c); // 执行实现函数
    duration = ustime()-start; // 计算命令执行耗费的时间
    dirty = server.dirty-dirty;
    if (dirty < 0) dirty = 0;
```

命令实现函数会将命令回复保存到客户端的输出缓冲区里面，并为客户端的套接字关联命令回复处理器，当客户端套接字变为可写状态时， 服务器就会执行命令回复处理器，将保存在客户端输出缓冲区中的命令回复发送给客户端。
当命令回复发送完毕之后，回复处理器会清空客户端状态的输出缓冲区，为处理下一个命令请求做好准备。

## 命令结构体
```c
struct redisCommand {
    // 命令名称
    char *name;
    // 实现函数
    redisCommandProc *proc;
    // 参数个数
    int arity;
    // 字符串表示的 FLAG
    char *sflags;   /* Flags as string representation, one char per flag. */
    // 实际 FLAG
    uint64_t flags; /* The actual flags, obtained from the 'sflags' field. */
    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    // 从命令中判断命令的键参数。在 Redis 集群转向时使用。
    redisGetKeysProc *getkeys_proc;
    /* What keys should be loaded in background when calling this command? */
    // 指定哪些参数是 key
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */
    // 统计信息
    // microseconds 记录了命令执行耗费的总毫微秒数
    // calls 是命令被执行的总次数
    long long microseconds, calls;
    int id;     /* Command ID. This is a progressive ID starting from 0 that
                   is assigned at runtime, and is used in order to check
                   ACLs. A connection is able to execute a given command if
                   the user associated to the connection has this command
                   bit set in the bitmap of allowed commands. */
};
```

## 命令执行
```c
/* SET key value [NX] [XX] [KEEPTTL] [EX <seconds>] [PX <milliseconds>] */
void setCommand(client *c) {
    ...
    c->argv[2] = tryObjectEncoding(c->argv[2]);
    setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}

void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
    ...
    genericSetKey(c,c->db,key,val,flags & OBJ_SET_KEEPTTL,1);
    server.dirty++;
    if (expire) setExpire(c,c->db,key,mstime()+milliseconds);
    notifyKeyspaceEvent(NOTIFY_STRING,"set",key,c->db->id);
    if (expire) notifyKeyspaceEvent(NOTIFY_GENERIC,
        "expire",key,c->db->id);
    addReply(c, ok_reply ? ok_reply : shared.ok);
}

代码：db.c
void genericSetKey(client *c, redisDb *db, robj *key, robj *val, int keepttl, int signal) {
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        dbOverwrite(db,key,val);
    }
    incrRefCount(val);
    if (!keepttl) removeExpire(db,key);
    if (signal) signalModifiedKey(c,db,key);
}
```

## 数据库
Redis 中的每个数据库，都由一个redisDb结构表示
```c
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    // 保存着数据库中的所有键值对数据
    dict *dict;                 /* The keyspace for this DB */
    // 保存着键的过期信息
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    // 用于实现 WATCH 命令
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```
### 添加键
添加一个新键对到数据库，实际上就是将一个新的键值对添加到键空间字典中，其中键为字符串对象，而值则是任意一种Redis类型值对象。  

### 删除键
删除数据库中的一个键，实际上就是删除字典空间中对应的键对象和值对象。  

### 更新键
当对一个已存在于数据库的键执行更新操作时，数据库释放键原来的值对象，然后将指针指向新的值对象。 

### 取值
在数据库中取值实际上就是在字典空间中取值，再加上一些额外的类型检查：  
1.键不存在，返回空回复。  
2.键存在，且类型正确，按照通讯协议返回值对象。  
3.键存在，但类型不正确，返回类型错误。  

### 过期时间
数据库中，所有键的过期时间都被保存在redisDb结构的expires字典里。
expires字典的键是一个指向dict字典（键空间）里某个键的指针，而字典的值则是键所指向的数据库键的到期时间，这个值以long long类型表示。

### 删除策略
Redis使用的过期键删除策略是惰性删除（取出键值时，要检查键是否过期）加上定期删除，这两个策略相互配合，可以很好地在合理利用CPU时间和节约内存空间之间取得平衡。

### 惰性删除
```c
int expireIfNeeded(redisDb *db, robj *key) {
    if (!keyIsExpired(db,key)) return 0;

    if (server.masterhost != NULL) return 1;

    /* Delete the key */
    server.stat_expiredkeys++;
    // 向AOF和从节点传播过期消息
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    // 发送键空间通知
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    int retval = server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                               dbSyncDelete(db,key);
    if (retval) signalModifiedKey(NULL,db,key);
    return retval;
}

/* Check if the key is expired. */
int keyIsExpired(redisDb *db, robj *key) {
    mstime_t when = getExpire(db,key);
    mstime_t now;

    if (when < 0) return 0; /* No expire for this key */
    /* Don't expire anything while loading. It will be done later. */
    if (server.loading) return 0;
  
    if (server.lua_caller) {
        now = server.lua_time_start;
    }
    else if (server.fixed_time_expire > 0) {
        now = server.mstime;
    }
    /* For the other cases, we want to use the most fresh time we have. */
    else {
        now = mstime();
    }

    /* The key expired if the current (virtual or real) time is greater
     * than the expire time of the key. */
    return now > when;
}
```

### 定期删除
serverCron ==>  databasesCron  ==> activeExpireCycle
```c
struct redisServer {
    int hz;                     /* serverCron() calls frequency in hertz */
}

void databasesCron(void) {
    /* Expire keys by random sampling. Not required for slaves
     * as master will synthesize DELs for us. */
    if (server.active_expire_enabled) {
        if (iAmMaster()) {
            activeExpireCycle(ACTIVE_EXPIRE_CYCLE_SLOW);
        } else {
            expireSlaveKeys();
        }
    }
    ...
}
```

### 空间的收缩和扩展
因为数据库空间是由字典来实现的，所以数据库空间的扩展/收缩规则和字典的扩展/收缩规则完全一样。

```c
void databasesCron(void) {
    ...
    /* Resize */
    for (j = 0; j < dbs_per_call; j++) {
        tryResizeHashTables(resize_db % server.dbnum);
        resize_db++;
    }
    ...
}

/* If the percentage of used slots in the HT reaches HASHTABLE_MIN_FILL
 * we resize the hash table to save memory */
void tryResizeHashTables(int dbid) {
    // 缩小键空间字典
    if (htNeedsResize(server.db[dbid].dict))
        dictResize(server.db[dbid].dict);
    // 缩小过期时间字典
    if (htNeedsResize(server.db[dbid].expires))
        dictResize(server.db[dbid].expires);
}
```

## RDB
Redis 分别提供了 RDB 和 AOF 两种持久化机制：

RDB 将数据库的快照（snapshot）以二进制的方式保存到磁盘中。
AOF 则以协议文本的方式，将所有对数据库进行过写入的命令（及其参数）记录到 AOF 文件，以此达到记录数据库状态的目的。  

RDB优点：
1 适合大规模的数据恢复。
2 如果业务对数据完整性和一致性要求不高，RDB是很好的选择。

内存数据对象 ----rdbSave()----> RDB文件
RDB文件 ----rdbLoad()----> 内存数据对象

RDB 文件结构:
```
+-------+-------------+-----------+-----------------+-----+-----------+
| REDIS | RDB-VERSION | SELECT-DB | KEY-VALUE-PAIRS | EOF | CHECK-SUM |
+-------+-------------+-----------+-----------------+-----+-----------+

                      |<-------- DB-DATA ---------->|
```

## AOF
除了RDB持久化功能以外，Redis还提供了AOF(Append Only File)持久化功能。与RDB持久化通过保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis所执行的写命令来记录数据库状态的。

文件写入和保存
每当服务器常规任务函数被执行、或者事件处理器被执行时，aof.c/flushAppendOnlyFile函数都会被调用，这个函数执行以下两个工作：  
WRITE：根据条件，将aof_buf中的缓存写入到AOF文件。  
SAVE：根据条件，调用fsync或fdatasync函数，将AOF文件保存到磁盘中。  
两个步骤都需要根据一定的条件来执行，而这些条件由AOF所使用的保存模式来决定。

Redis目前支持三种 AOF 保存模式，它们分别是：  
AOF_FSYNC_NO ：不保存。  
AOF_FSYNC_EVERYSEC ：每一秒钟保存一次。  
AOF_FSYNC_ALWAYS ：每执行一个命令保存一次。  

AOF重写的作用:  
1.减少磁盘占用量  
2.加速恢复速度  

触发条件:  
1.调用BGREWRITEAOF手动触发  
2.当serverCron函数执行时，自动触发

## 发布/订阅
当一个客户端通过 PUBLISH 命令向订阅者发送信息的时候，我们称这个客户端为发布者（publisher）。
而当一个客户端使用 SUBSCRIBE 或者 PSUBSCRIBE 命令接收信息的时候，我们称这个客户端为订阅者（subscriber）。
为了解耦发布者（publisher）和订阅者（subscriber）之间的关系，Redis使用了channel（频道）作为两者的中介 —— 发布者将信息直接发布给channel，而channel负责将信息发送给适当的订阅者，发布者和订阅者之间没有相互关系，也不知道对方的存在。

### SUBSCRIBE命令
Redis将所有接受和发送信息的任务交给channel来进行，而所有channel的信息就储存在redisServer这个结构中。  
```c
struct redisServer {
    ...
    /* Pubsub */
    dict *pubsub_channels;  /* Map channels to list of subscribed clients */
    ...
}
```
pubsub_channels是一个字典，字典的键就是一个个channel ，而字典的值则是一个链表，链表中保存了所有订阅这个channel的客户端。

函数pubsubSubscribeChannel是SUBSCRIBE命令的底层实现，它完成了将客户端添加到订阅链表中的工作
```c
// 订阅指定频道
// 订阅成功返回 1 ，如果已经订阅过，返回 0
/* Subscribe a client to a channel. Returns 1 if the operation succeeded, or
 * 0 if the client was already subscribed to that channel. */
int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* Add the channel to the client -> channels hash table */
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        // 订阅成功
        retval = 1;
        incrRefCount(channel);
        // 将 client 添加到订阅给定 channel 的链表中
        /* Add the client to the channel -> list of clients hash table */
        de = dictFind(server.pubsub_channels,channel);
        if (de == NULL) {
            // 如果 de 等于 NULL
            // 表示这个客户端是首个订阅这个 channel 的客户端
            // 那么创建一个新的列表， 并将它加入到哈希表中
            clients = listCreate();
            dictAdd(server.pubsub_channels,channel,clients);
            incrRefCount(channel);
        } else {
            // 如果 de 不为空，就取出这个 clients 链表
            clients = dictGetVal(de);
        }
        // 将客户端加入到链表中
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    // 返回客户端当前已订阅的频道和订阅数量
    addReplyPubsubSubscribed(c,channel);
    return retval;
}

```

### PSUBSCRIBE命令
除了直接订阅给定channel外，还可以使用PSUBSCRIBE订阅一个模式（pattern），订阅一个模式等同于订阅所有匹配这个模式的 channel。
redisServer.pubsub_patterns属性用于保存所有被订阅的模式，和pubsub_channels不同的是， pubsub_patterns是一个链表（而不是字典）
```c
struct redisServer {
    ...
    list *pubsub_patterns;  /* A list of pubsub_patterns */
    dict *pubsub_patterns_dict;  /* A dict of pubsub_patterns */
    ...
}
```
pubsubSubscribePattern是PSUBSCRIBE的底层实现，它将客户端和所订阅的模式添加到redisServer.pubsub_patterns当中：
```c
/* Subscribe a client to a pattern. Returns 1 if the operation succeeded, or 0 if the client was already subscribed to that pattern. */
int pubsubSubscribePattern(client *c, robj *pattern) {
    ...
    if (listSearchKey(c->pubsub_patterns,pattern) == NULL) {
        retval = 1;
        pubsubPattern *pat;
        // 添加 pattern 到客户端 pubsub_patterns
        listAddNodeTail(c->pubsub_patterns,pattern);
        incrRefCount(pattern);
        pat = zmalloc(sizeof(*pat));
        pat->pattern = getDecodedObject(pattern);
        pat->client = c;
        // 将 pattern 添加到服务器
        listAddNodeTail(server.pubsub_patterns,pat);
        /* Add the client to the pattern -> list of clients hash table */
        de = dictFind(server.pubsub_patterns_dict,pattern);
        if (de == NULL) {
            clients = listCreate();
            dictAdd(server.pubsub_patterns_dict,pattern,clients);
            incrRefCount(pattern);
        } else {
            clients = dictGetVal(de);
        }
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    addReplyPubsubPatSubscribed(c,pattern);
    return retval;
}

```

### PUBLISH命令
使用 PUBLISH 命令向订阅者发送消息，需要执行以下两个步骤：  
1.使用给定的频道作为键，在 redisServer.pubsub_channels 字典中查找记录了订阅这个频道的所有客户端的链表，遍历这个链表，将消息发布给所有订阅者。  
2.遍历 redisServer.pubsub_patterns 链表，将链表中的模式和给定的频道进行匹配，如果匹配成功，那么将消息发布到相应模式的客户端当中。  

```
/* Publish a message */
int pubsubPublishMessage(robj *channel, robj *message) {
    ...
    /* Send to clients listening for that channel */
    // 向所有频道的订阅者发送消息
    de = dictFind(server.pubsub_channels,channel);
    if (de) {
        list *list = dictGetVal(de); // 取出所有订阅者
        listNode *ln;
        listIter li;
        // 遍历所有订阅者， 向它们发送消息
        listRewind(list,&li);
        while ((ln = listNext(&li)) != NULL) {
            client *c = ln->value;
            addReplyPubsubMessage(c,channel,message);
            receivers++;  // 更新接收者数量
        }
    }
    // 向所有被匹配模式的订阅者发送消息
    /* Send to clients listening to matching channels */
    di = dictGetIterator(server.pubsub_patterns_dict);
    if (di) {
        channel = getDecodedObject(channel);
        while((de = dictNext(di)) != NULL) {
            robj *pattern = dictGetKey(de);
            list *clients = dictGetVal(de);
            // 如果模式和channel匹配的话
            // 向这个模式的订阅者发送消息
            if (!stringmatchlen((char*)pattern->ptr,
                                sdslen(pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) continue;

            listRewind(clients,&li);
            while ((ln = listNext(&li)) != NULL) {
                client *c = listNodeValue(ln);
                addReplyPubsubPatMessage(c,pattern,channel,message);
                receivers++;
            }
        }
        decrRefCount(channel);
        dictReleaseIterator(di);
    }
    return receivers;// 返回接收者数量
}
```

## 事务
Redis的事务使用MULTI命令和EXEC命令包围，处在这两条命令之间的一条或多条命令，会以FIFO的方式运行。
Redis的事务和关系数据库的事务并不一样，Redis的事务并不保证 ACID性质。

### MULTI命令
```c
void multiCommand(client *c) {
    // MULTI 不可以嵌套使用
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
     // 打开 FLAG
    c->flags |= CLIENT_MULTI;
    addReply(c,shared.ok);
}
```
当CLIENT_MULTI这个FLAG被打开之后，传入 Redis客户端的命令就不会马上被执行（部分命令如 EXEC 除外），这些未被执行的命令会被queueMultiCommand以FIFO的方式放入一个数组里，储存起来。
```c
int processCommand(client *c) {
    ...
    /* Exec the command */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        // 在事务上下文中除 EXEC 、 DISCARD 、 MULTI 和 WATCH 命令之外
        // 其他所有命令都会被入队到事务队列中
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        // 执行命令
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            // 处理那些解除了阻塞的键
            handleClientsBlockedOnKeys();
    }
    return C_OK;
}
```
queueMultiCommand函数将要执行的命令、命令的参数个数以及命令的参数放进multiCmd结构中，并将这个结构保存到redisClient.mstate.command数组的末尾，从而形成一个保存了要执行的命令的FIFO队列
```c
/* Add a new command into the MULTI commands queue */
void queueMultiCommand(client *c) {
    multiCmd *mc;
    int j;

    /* No sense to waste memory if the transaction is already aborted.
     * this is useful in case client sends these in a pipeline, or doesn't
     * bother to read previous responses and didn't notice the multi was already
     * aborted. */
    if (c->flags & CLIENT_DIRTY_EXEC)
        return;
    // 为新命令分配储存结构，并放到数组的末尾
    c->mstate.commands = zrealloc(c->mstate.commands,
            sizeof(multiCmd)*(c->mstate.count+1));
    // 设置新命令
    mc = c->mstate.commands+c->mstate.count; // 指向储存新命令的结构体
    mc->cmd = c->cmd;                        // 设置命令
    mc->argc = c->argc;                      // 设置参数数量
    mc->argv = zmalloc(sizeof(robj*)*c->argc);      // 生成参数空间
    memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc); // 设置参数
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    c->mstate.count++;  // 更新命令数量的计数器
    c->mstate.cmd_flags |= c->cmd->flags;
    c->mstate.cmd_inv_flags |= ~c->cmd->flags;
}
```

### 执行事务
事务的执行由execCommand函数进行。
```c
void execCommand(client *c) {
    ...
    // 如果没执行过 MULTI ，报错
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"EXEC without MULTI");
        return;
    }
    /* Check if we need to abort the EXEC because:
     * 1) Some WATCHed key was touched.
     * 2) There was a previous error while queueing commands.
     * A failed EXEC in the first case returns a multi bulk nil object
     * (technically it is not an error but a special behavior), while
     * in the second an EXECABORT error is returned. */
    if (c->flags & (CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)) {
        addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
                                                   shared.nullarray[c->resp]);
        discardTransaction(c);
        goto handle_monitor;
    }

    /* Exec all the queued commands */
    unwatchAllKeys(c); /* Unwatch ASAP otherwise we'll waste CPU cycles */
    // 备份所有参数和命令
    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyArrayLen(c,c->mstate.count);
    for (j = 0; j < c->mstate.count; j++) {
        c->argc = c->mstate.commands[j].argc; // 取出参数数量
        c->argv = c->mstate.commands[j].argv; // 取出参数
        c->cmd = c->mstate.commands[j].cmd;   // 取出要执行的命令

        /* Propagate a MULTI request once we encounter the first command which
         * is not readonly nor an administrative one.
         * This way we'll deliver the MULTI/..../EXEC block as a whole and
         * both the AOF and the replication link will have the same consistency
         * and atomicity guarantees. */
        if (!must_propagate &&
            !server.loading &&
            !(c->cmd->flags & (CMD_READONLY|CMD_ADMIN)))
        {
            // 如果处在 AOF 模式中，向 AOF 文件发送 MULTI
            // 如果处在复制模式中，向附属节点发送 MULTI
            execCommandPropagateMulti(c);
            must_propagate = 1;
        }
        // ACL（Access Control List),判断权限
        int acl_keypos;
        int acl_retval = ACLCheckCommandPerm(c,&acl_keypos);
        if (acl_retval != ACL_OK) {
            ...
        } else {
            // 执行命令
            call(c,server.loading ? CMD_CALL_NONE : CMD_CALL_FULL);
        }

        /* Commands may alter argc/argv, restore mstate. */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }
    // 恢复所有参数和命令
    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    discardTransaction(c);

    /* Make sure the EXEC command will be propagated as well if MULTI
     * was already propagated. */
    if (must_propagate) {
        int is_master = server.masterhost == NULL;
        server.dirty++;
        /* If inside the MULTI/EXEC block this instance was suddenly
         * switched from master to slave (using the SLAVEOF command), the
         * initial MULTI was propagated into the replication backlog, but the
         * rest was not. We need to make sure to at least terminate the
         * backlog with the final EXEC. */
        if (server.repl_backlog && was_master && !is_master) {
            char *execcmd = "*1\r\n$4\r\nEXEC\r\n";
            feedReplicationBacklog(execcmd,strlen(execcmd));
        }
    }

handle_monitor:
    /* Send EXEC to clients waiting data from MONITOR. We do it here
     * since the natural order of commands execution is actually:
     * MUTLI, EXEC, ... commands inside transaction ...
     * Instead EXEC is flagged as CMD_SKIP_MONITOR in the command
     * table, and we do it here with correct ordering. */
    if (listLength(server.monitors) && !server.loading)
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
}
```

### 取消事务
DISCARD命令是用来中途取消事务的，由discardCommand和discardTransaction两个函数实现。
```c
void discardCommand(client *c) {
    // 如果没有调用过 MULTI ，报错
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"DISCARD without MULTI");
        return;
    }
    discardTransaction(c);
    addReply(c,shared.ok);
}

void discardTransaction(client *c) {
    freeClientMultiState(c);  // 释放事务资源
    initClientMultiState(c);  // 重置事务状态
    c->flags &= ~(CLIENT_MULTI|CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC); // 关闭 FLAG
    unwatchAllKeys(c);   // 取消对所有 key 的 WATCH
}
```
其中freeClientMultiState和initClientMultiState两个函数用于重置client.mstate数组，从而达到删除所有入队命令的作用。
```c
void freeClientMultiState(client *c) {
    int j;
    // 释放所有命令
    for (j = 0; j < c->mstate.count; j++) {
        int i;
        multiCmd *mc = c->mstate.commands+j;
        // 释放所有命令的参数，以及保存参数的数组
        for (i = 0; i < mc->argc; i++)
            decrRefCount(mc->argv[i]);
        zfree(mc->argv);
    }
    // 释放保存命令的数组
    zfree(c->mstate.commands);
}
void initClientMultiState(client *c) {
    c->mstate.commands = NULL;  // 清空命令数组
    c->mstate.count = 0;        // 清空命令计数器
    c->mstate.cmd_flags = 0;
    c->mstate.cmd_inv_flags = 0;
}
```