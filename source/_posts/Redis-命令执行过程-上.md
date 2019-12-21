---
title: Redis 命令执行过程(上)
tags: redis
abbrlink: 4cc1fb9
date: 2019-12-09 22:25:54
---
今天我们来了解一下 Redis 命令执行的过程。在之前的文章中[《当 Redis 发生高延迟时，到底发生了什么》](http://remcarpediem.net/article/ef4e619/)我们曾简单的描述了一条命令的执行过程，本篇文章展示深入说明一下，加深读者对 Redis 的了解。

如下图所示，一条命令执行完成并且返回数据一共涉及三部分，第一步是建立连接阶段，响应了socket的建立，并且创建了client对象；第二步是处理阶段，从socket读取数据到输入缓冲区，然后解析并获得命令，执行命令并将返回值存储到输出缓冲区中；第三步是数据返回阶段，将返回值从输出缓冲区写到socket中，返回给客户端，最后关闭client。

![](/images/19_1221/1_image1.png)

这三个阶段之间是通过事件机制串联了，在 Redis 启动阶段首先要注册socket连接建立事件处理器：
- 当客户端发来建立socket的连接的请求时，对应的处理器方法会被执行，建立连接阶段的相关处理就会进行，然后注册socket读取事件处理器
- 当客户端发来命令时，读取事件处理器方法会被执行，对应处理阶段的相关逻辑都会被执行，然后注册socket写事件处理器
- 当写事件处理器被执行时，就是将返回值写回到socket中。


![](/images/19_1221/2_image1.png)

接下来，我们分别来看一下各个步骤的具体原理和代码实现。

### 启动时监听socket

Redis 服务器启动时，会调用 initServer 方法，首先会建立 Redis 自己的事件机制 eventLoop，然后在其上注册周期时间事件处理器，最后在所监听的 socket 上
创建文件事件处理器，监听 socket 建立连接的事件，其处理函数为 acceptTcpHandler。
``` c
void initServer(void) { // server.c
    ....
    /**
     * 创建eventLoop
     */
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
    /* Open the TCP listening socket for the user commands. */

    if (server.port != 0 &&
        listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)
        exit(1);

    /**
     * 注册周期时间事件，处理后台操作，比如说客户端操作、过期键等
     */
    if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
        serverPanic("Can't create event loop timers.");
        exit(1);
    }
    /**
     * 为所有监听的socket创建文件事件，监听可读事件；事件处理函数为acceptTcpHandler
     * 
     */
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
    ....
}
```
在《Redis 事件机制详解》一文中，我们曾详细介绍过 Redis 的事件机制，可以说，Redis 命令执行过程中都是由事件机制协调管理的，也就是 initServer 方法中生成的 aeEventLoop。当socket发生对应的事件时，aeEventLoop 对调用已经注册的对应的事件处理器。

![](/images/19_1221/2_image2.png)

### 建立连接和Client

当客户端向 Redis 建立 socket时，aeEventLoop 会调用 acceptTcpHandler 处理函数，服务器会为每个链接创建一个 Client 对象，并创建相应文件事件来监听socket的可读事件，并指定事件处理函数。

acceptTcpHandler 函数会首先调用 `anetTcpAccept `方法，它底层会调用 socket 的 accept 方法，也就是接受客户端来的建立连接请求，然后调用 `acceptCommonHandler `方法，继续后续的逻辑处理。
```
// 当客户端建立链接时进行的eventloop处理函数  networking.c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    ....
    // 层层调用，最后在anet.c 中 anetGenericAccept 方法中调用 socket 的 accept 方法
    cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
    if (cfd == ANET_ERR) {
        if (errno != EWOULDBLOCK)
            serverLog(LL_WARNING,
                "Accepting client connection: %s", server.neterr);
        return;
    }
    serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
    /**
     * 进行socket 建立连接后的处理
     */
    acceptCommonHandler(cfd,0,cip);
}
```

acceptCommonHandler 则首先调用 createClient 创建 client，接着判断当前 client 的数量是否超出了配置的 maxclients，如果超过，则给客户端发送错误信息，并且释放 client。

``` c
static void acceptCommonHandler(int fd, int flags, char *ip) { //networking.c
    client *c;
    // 创建redisClient
    c = createClient(fd)
    // 当 maxClient 属性被设置，并且client数量已经超出时，给client发送error，然后释放连接
    if (listLength(server.clients) > server.maxclients) {
        char *err = "-ERR max number of clients reached\r\n";
        if (write(c->fd,err,strlen(err)) == -1) {
        }
        server.stat_rejected_conn++;
        freeClient(c);
        return;
    }
    .... // 处理为设置密码时默认保护状态的客户端连接
    // 统计连接数
    server.stat_numconnections++;
    c->flags |= flags;
}
```

createClient 方法用于创建 client，它代表着连接到 Redis 客户端，每个客户端都有各自的输入缓冲区和输出缓冲区，输入缓冲区存储客户端通过 socket 发送过来的数据，输出缓冲区则存储着 Redis 对客户端的响应数据。client一共有三种类型，不同类型的对应缓冲区的大小都不同。

- 普通客户端是除了复制和订阅的客户端之外的所有连接
- 从客户端用于主从复制，主节点会为每个从节点单独建立一条连接用于命令复制
- 订阅客户端用于发布订阅功能

![](/images/19_1221/2_image3.png)


createClient 方法除了创建 client 结构体并设置其属性值外，还会对 socket进行配置并注册读事件处理器

设置 socket 为 非阻塞 socket、设置 NO_DELAY 和 SO_KEEPALIVE标志位来关闭 Nagle 算法并且启动 socket 存活检查机制。

设置读事件处理器，当客户端通过 socket 发送来数据后，Redis 会调用 readQueryFromClient 方法。

```
client *createClient(int fd) {
    client *c = zmalloc(sizeof(client));
    // fd 为 -1，表示其他特殊情况创建的client，redis在进行比如lua脚本执行之类的情况下也会创建client
    if (fd != -1) {
        // 配置socket为非阻塞、NO_DELAY不开启Nagle算法和SO_KEEPALIVE
        anetNonBlock(NULL,fd);
        anetEnableTcpNoDelay(NULL,fd);
        if (server.tcpkeepalive)
            anetKeepAlive(NULL,fd,server.tcpkeepalive);
        /**
         * 向 eventLoop 中注册了 readQueryFromClient。
         * readQueryFromClient 的作用就是从client中读取客户端的查询缓冲区内容。
         * 绑定读事件到事件 loop （开始接收命令请求）
         */
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        }
    }
    // 默认选择数据库
    selectDb(c,0);
    uint64_t client_id;
    atomicGetIncr(server.next_client_id,client_id,1);
    c->id = client_id;
    c->fd = fd;
    .... // 设置client的属性
    return c;
}
```

client 的属性中有很多属性，比如后边会看到的输入缓冲区 querybuf 和输出缓冲区 buf，这里因为代码过长做了省略，感兴趣的同学可以自行阅读源码。

### 读取socket数据到输入缓冲区

readQueryFromClient 方法会调用 read 方法从 socket 中读取数据到输入缓冲区中，然后判断其大小是否大于系统设置的 client_max_querybuf_len，如果大于，则向 Redis返回错误信息，并关闭 client。

将数据读取到输入缓冲区后，readQueryFromClient 方法会根据 client 的类型来做不同的处理，如果是普通类型，则直接调用 processInputBuffer 来处理；如果是主从客户端，还需要将命令同步到自己的从服务器中。也就是说，Redis实例将主实例传来的命令执行后，继续将命令同步给自己的从实例。

![](/images/19_1221/2_image4.png)


```
// 处理从client中读取客户端的输入缓冲区内容。
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata;
    ....
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    // 从 fd 对应的socket中读取到 client 中的 querybuf 输入缓冲区
    nread = read(fd, c->querybuf+qblen, readlen);
    if (nread == -1) {
        .... // 出错释放 client
    } else if (nread == 0) {
        // 客户端主动关闭 connection
        serverLog(LL_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    } else if (c->flags & CLIENT_MASTER) { 
        /*
         * 当这个client代表主从的master节点时，将query buffer和 pending_querybuf结合
         * 用于主从复制中的命令传播？？？？
         */
        c->pending_querybuf = sdscatlen(c->pending_querybuf,
                                        c->querybuf+qblen,nread);
    }
    // 增加已经读取的字节数
    sdsIncrLen(c->querybuf,nread);
    c->lastinteraction = server.unixtime;
    if (c->flags & CLIENT_MASTER) c->read_reploff += nread;
    server.stat_net_input_bytes += nread;
    // 如果大于系统配置的最大客户端缓存区大小，也就是配置文件中的client-query-buffer-limit
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();
        // 返回错误信息，并且关闭client
        bytes = sdscatrepr(bytes,c->querybuf,64);
        serverLog(LL_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
        sdsfree(ci);
        sdsfree(bytes);
        freeClient(c);
        return;
    }

    
    if (!(c->flags & CLIENT_MASTER)) {
        // processInputBuffer 处理输入缓冲区
        processInputBuffer(c);
    } else {
        // 如果client是master的连接
        size_t prev_offset = c->reploff;
        processInputBuffer(c);
        // 判断是否同步偏移量发生变化，则通知到后续的slave
        size_t applied = c->reploff - prev_offset;

        if (applied) {
            replicationFeedSlavesFromMasterStream(server.slaves,
                    c->pending_querybuf, applied);
            sdsrange(c->pending_querybuf,applied,-1);
        }
    }
}
```

### 解析获取命令

processInputBuffer 主要是将输入缓冲区中的数据解析成对应的命令，根据命令类型是 PROTO_REQ_MULTIBULK 还是 PROTO_REQ_INLINE，来分别调用 processInlineBuffer 和 processMultibulkBuffer 方法来解析命令。

然后调用 processCommand 方法来执行命令。执行成功后，如果是主从客户端，还需要更新同步偏移量 reploff 属性，然后重置 client，让client可以接收一条命令。

```
void processInputBuffer(client *c) { // networking.c
    server.current_client = c;
    /* 当缓冲区中还有数据时就一直处理 */
    while(sdslen(c->querybuf)) {
        .... // 处理 client 的各种状态
        /* 判断命令请求类型 telnet发送的命令和redis-cli发送的命令请求格式不同 */
        if (!c->reqtype) {
            if (c->querybuf[0] == '*') {
                c->reqtype = PROTO_REQ_MULTIBULK;
            } else {
                c->reqtype = PROTO_REQ_INLINE;
            }
        }
        /**
         * 从缓冲区解析命令
         */
        if (c->reqtype == PROTO_REQ_INLINE) {
            if (processInlineBuffer(c) != C_OK) break;
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != C_OK) break;
        } else {
            serverPanic("Unknown request type");
        }

        /* 参数个数为0时重置client，可以接受下一个命令 */
        if (c->argc == 0) {
            resetClient(c);
        } else {
            // 执行命令
            if (processCommand(c) == C_OK) {
                if (c->flags & CLIENT_MASTER && !(c->flags & CLIENT_MULTI)) {
                    // 如果是master的client发来的命令，则 更新 reploff
                    c->reploff = c->read_reploff - sdslen(c->querybuf);
                }

                // 如果不是阻塞状态，则重置client，可以接受下一个命令
                if (!(c->flags & CLIENT_BLOCKED) || c->btype != BLOCKED_MODULE)
                    resetClient(c);
            }
        }
    }
    server.current_client = NULL;
}
```

解析命令暂时不看，就是将 redis 命令文本信息，记录到client的argv/argc属性中

### 执行命令

processCommand 方法会处理很多逻辑，不过大致可以分为三个部分：首先是调用 lookupCommand 方法获得对应的 redisCommand；接着是检测当前 Redis 是否可以执行该命令；最后是调用 call 方法真正执行命令。

processCommand会做如下逻辑处理：
- 1 如果命令名称为 quit，则直接返回，并且设置客户端标志位。
- 2 根据 argv[0] 查找对应的 redisCommand，所有的命令都存储在命令字典 redisCommandTable 中，根据命令名称可以获取对应的命令。
- 3 进行用户权限校验。
- 4 如果是集群模式，处理集群重定向。当命令发送者是 master 或者 命令没有任何 key 的参数时可以不重定向。
- 5 预防 maxmemory 情况，先尝试回收一下，如果不行，则返回异常。
- 6 当此服务器是 master 时：aof 持久化失败时，或上一次 bgsave 执行错误，且配置 bgsave 参数和 stop_writes_on_bgsave_err；禁止执行写命令。
- 7 当此服务器时master时：如果配置了 repl_min_slaves_to_write，当slave数目小于时，禁止执行写命令。
- 8 当时只读slave时，除了 master 的不接受其他写命令。
- 9 当客户端正在订阅频道时，只会执行部分命令。
- 10 服务器为slave，但是没有连接 master 时，只会执行带有 CMD_STALE 标志的命令，如 info 等
- 11 正在加载数据库时，只会执行带有 CMD_LOADING 标志的命令，其余都会被拒绝。
- 12 当服务器因为执行lua脚本阻塞时，只会执行部分命令，其余都会拒绝
- 13 如果是事务命令，则开启事务，命令进入等待队列；否则直接执行命令。


``` c
int processCommand(client *c) {
    // 1 处理 quit 命令
    if (!strcasecmp(c->argv[0]->ptr,"quit")) {
        addReply(c,shared.ok);
        c->flags |= CLIENT_CLOSE_AFTER_REPLY;
        return C_ERR;
    }

    /**
     * 根据 argv[0] 查找对应的 command
     * 2 命令字典查找指定命令；所有的命令都存储在命令字典中 struct redisCommand redisCommandTable[]={}
     */
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    if (!c->cmd) {
        // 处理未知命令
    } else if ((c->cmd->arity > 0 && c->cmd->arity != c->argc) ||
               (c->argc < -c->cmd->arity)) {
        // 处理参数错误
    }
    // 3 检查用户验证
    if (server.requirepass && !c->authenticated && c->cmd->proc != authCommand)
    {
        flagTransaction(c);
        addReply(c,shared.noautherr);
        return C_OK;
    }

    /**
     * 4 如果是集群模式，处理集群重定向。当命令发送者是master或者 命令没有任何key的参数时可以不重定向
     */
    if (server.cluster_enabled &&
        !(c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_LUA &&
          server.lua_caller->flags & CLIENT_MASTER) &&
        !(c->cmd->getkeys_proc == NULL && c->cmd->firstkey == 0 &&
          c->cmd->proc != execCommand))
    {
        int hashslot;
        int error_code;
        // 查询可以执行的node信息
        clusterNode *n = getNodeByQuery(c,c->cmd,c->argv,c->argc,
                                        &hashslot,&error_code);
        if (n == NULL || n != server.cluster->myself) {
            if (c->cmd->proc == execCommand) {
                discardTransaction(c);
            } else {
                flagTransaction(c);
            }
            clusterRedirectClient(c,n,hashslot,error_code);
            return C_OK;
        }
    }

    // 5 处理maxmemory请求，先尝试回收一下，如果不行，则返回异常
    if (server.maxmemory) {
        int retval = freeMemoryIfNeeded();
        ....
    }

    /**
     * 6 当此服务器是master时：aof持久化失败时，或上一次bgsave执行错误，
     * 且配置bgsave参数和stop_writes_on_bgsave_err；禁止执行写命令
     */
    if (((server.stop_writes_on_bgsave_err &&
          server.saveparamslen > 0 &&
          server.lastbgsave_status == C_ERR) ||
          server.aof_last_write_status == C_ERR) &&
        server.masterhost == NULL &&
        (c->cmd->flags & CMD_WRITE ||
         c->cmd->proc == pingCommand)) { .... }

    /**
     * 7 当此服务器时master时：如果配置了repl_min_slaves_to_write，
     * 当slave数目小于时，禁止执行写命令
     */
    if (server.masterhost == NULL &&
        server.repl_min_slaves_to_write &&
        server.repl_min_slaves_max_lag &&
        c->cmd->flags & CMD_WRITE &&
        server.repl_good_slaves_count < server.repl_min_slaves_to_write) { .... }

    /**
     * 8 当时只读slave时，除了master的不接受其他写命令
     */
    if (server.masterhost && server.repl_slave_ro &&
        !(c->flags & CLIENT_MASTER) &&
        c->cmd->flags & CMD_WRITE) { .... }

    /**
     * 9 当客户端正在订阅频道时，只会执行以下命令
     */
    if (c->flags & CLIENT_PUBSUB &&
        c->cmd->proc != pingCommand &&
        c->cmd->proc != subscribeCommand &&
        c->cmd->proc != unsubscribeCommand &&
        c->cmd->proc != psubscribeCommand &&
        c->cmd->proc != punsubscribeCommand) { .... }
    /**
     * 10 服务器为slave，但没有正确连接master时，只会执行带有CMD_STALE标志的命令，如info等
     */
    if (server.masterhost && server.repl_state != REPL_STATE_CONNECTED &&
        server.repl_serve_stale_data == 0 &&
        !(c->cmd->flags & CMD_STALE)) {...}
    /**
     * 11 正在加载数据库时，只会执行带有CMD_LOADING标志的命令，其余都会被拒绝
     */
    if (server.loading && !(c->cmd->flags & CMD_LOADING)) { .... }
    /**
     * 12 当服务器因为执行lua脚本阻塞时，只会执行以下几个命令，其余都会拒绝
     */
    if (server.lua_timedout &&
          c->cmd->proc != authCommand &&
          c->cmd->proc != replconfCommand &&
        !(c->cmd->proc == shutdownCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'n') &&
        !(c->cmd->proc == scriptCommand &&
          c->argc == 2 &&
          tolower(((char*)c->argv[1]->ptr)[0]) == 'k')) {....}

    /**
     * 13 开始执行命令
     */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        /**
         * 开启了事务，命令只会入队列
         */
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        /**
         * 直接执行命令
         */
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnLists();
    }
    return C_OK;
}


struct redisCommand redisCommandTable[] = {
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"hmset",hsetCommand,-4,"wmF",0,NULL,1,1,1,0,0},
    .... // 所有的 redis 命令都有
}
```

call 方法是 Redis 中执行命令的通用方法，它会处理通用的执行命令的前置和后续操作。

![](/images/19_1221/2_image5.png)


- 如果有监视器 monitor，则需要将命令发送给监视器。
- 调用 redisCommand 的proc 方法，执行对应具体的命令逻辑。
- 如果开启了 CMD_CALL_SLOWLOG，则需要记录慢查询日志
- 如果开启了 CMD_CALL_STATS，则需要记录一些统计信息
- 如果开启了 CMD_CALL_PROPAGATE，则当 dirty大于0时，需要调用 propagate 方法来进行命令传播。

![](/images/19_1221/2_image6.png)

命令传播就是将命令写入 repl-backlog-buffer 缓冲中，并发送给各个从服务器中。

``` c
// 执行client中持有的 redisCommand 命令
void call(client *c, int flags) {
    /**
     * dirty记录数据库修改次数；start记录命令开始执行时间us；duration记录命令执行花费时间
     */
    long long dirty, start, duration;
    int client_old_flags = c->flags;

    /**
     * 有监视器的话，需要将不是从AOF获取的命令会发送给监视器。当然，这里会消耗时间
     */
    if (listLength(server.monitors) &&
        !server.loading &&
        !(c->cmd->flags & (CMD_SKIP_MONITOR|CMD_ADMIN)))
    {
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
    }
    ....
    /* Call the command. */
    dirty = server.dirty;
    start = ustime();
    // 处理命令，调用命令处理函数
    c->cmd->proc(c);
    duration = ustime()-start;
    dirty = server.dirty-dirty;
    if (dirty < 0) dirty = 0;

    .... // Lua 脚本的一些特殊处理

    /**
     * CMD_CALL_SLOWLOG 表示要记录慢查询日志
     */
    if (flags & CMD_CALL_SLOWLOG && c->cmd->proc != execCommand) {
        char *latency_event = (c->cmd->flags & CMD_FAST) ?
                              "fast-command" : "command";
        latencyAddSampleIfNeeded(latency_event,duration/1000);
        slowlogPushEntryIfNeeded(c,c->argv,c->argc,duration);
    }
    /**
     * CMD_CALL_STATS 表示要统计
     */
    if (flags & CMD_CALL_STATS) {
        c->lastcmd->microseconds += duration;
        c->lastcmd->calls++;
    }
    /**
     * CMD_CALL_PROPAGATE表示要进行广播命令
     */
    if (flags & CMD_CALL_PROPAGATE &&
        (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP)
    {
        int propagate_flags = PROPAGATE_NONE;
        /**
         * dirty大于0时，需要广播命令给slave和aof
         */
        if (dirty) propagate_flags |= (PROPAGATE_AOF|PROPAGATE_REPL);
        .... 
        /**
         * 广播命令，写如aof，发送命令到slave
         * 也就是传说中的传播命令
         */
        if (propagate_flags != PROPAGATE_NONE && !(c->cmd->flags & CMD_MODULE))
            propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags);
    }
    ....
}
```
由于文章篇幅问题，本篇文章就先讲到这里，后半部分在接下来的文章中进行讲解，欢迎大家继续关注。


![](/images/logo.png)