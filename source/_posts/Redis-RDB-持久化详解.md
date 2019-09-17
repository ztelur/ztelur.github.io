---
title: Redis RDB 持久化详解
tags: redis
abbrlink: 9a7bcda
date: 2019-07-04 21:28:23
---
Redis 是一种内存数据库，将数据保存在内存中，读写效率要比传统的将数据保存在磁盘上的数据库要快很多。但是一旦进程退出，Redis 的数据就会丢失。

为了解决这个问题，Redis 提供了 RDB 和 AOF 两种持久化方案，将内存中的数据保存到磁盘中，避免数据丢失。

antirez 在《Redis 持久化解密》一文中说，一般来说有三种常见的策略来进行持久化操作，防止数据损坏：

- 方法1 是数据库不关心发生故障，在数据文件损坏后通过数据备份或者快照来进行恢复。Redis 的 RDB 持久化就是这种方式。

- 方法2 是数据库使用操作日志，每次操作时记录操作行为，以便在故障后通过日志恢复到一致性的状态。因为操作日志是顺序追加的方式写的，所以不会出现操作日志也无法恢复的情况。类似于 Mysql 的 redo 和 undo 日志，具体可以看这篇[《InnoDB的磁盘文件及落盘机制》](https://mp.weixin.qq.com/s/QaN-ROOW06b6rm-HoiSX3g)文章。

- 方法3 是数据库不进行老数据的修改，只是以追加方式去完成写操作，这样数据本身就是一份日志，这样就永远不会出现数据无法恢复的情况了。CouchDB就是此做法的优秀范例。

RDB 就是第一种方法，它就是把当前 Redis 进程的数据生成时间点快照( point-in-time snapshot ) 保存到存储设备的过程。

### RDB 的使用

RDB 触发机制分为使用指令手动触发和 redis.conf 配置自动触发。

手动触发 Redis 进行 RDB 持久化的指令的为:
- save ，该指令会阻塞当前 Redis 服务器，执行 save 指令期间，Redis 不能处理其他命令，直到 RDB 过程完成为止。
- bgsave，执行该命令时，Redis 会在后台异步执行快照操作，此时 Redis 仍然可以相应客户端请求。具体操作是 Redis 进程执行 `fork` 操作创建子进程，RDB 持久化过程由子进程负责，完成后自动结束。Redis 只会在 `fork` 期间发生阻塞，但是一般时间都很短。但是如果 Redis 数据量特别大，`fork` 时间就会变长，而且占用内存会加倍，这一点需要特别注意。

自动触发 RDB 的默认配置如下所示:
```
save 900 1 # 表示900 秒内如果至少有 1 个 key 的值变化，则触发RDB
save 300 10 # 表示300 秒内如果至少有 10 个 key 的值变化，则触发RDB
save 60 10000 # 表示60 秒内如果至少有 10000 个 key 的值变化，则触发RDB
```

如果不需要 Redis 进行持久化，那么可以注释掉所有的 save 行来停用保存功能，也可以直接一个空字符串来停用持久化：save ""。

Redis 服务器周期操作函数 `serverCron` 默认每个 100 毫秒就会执行一次，该函数用于正在运行的服务器进行维护，它的一项工作就是检查 save 选项所设置的条件是否有一项被满足，如果满足的话，就执行 bgsave 指令。


### RDB 整体流程

了解了 RDB 的基础使用后，我们要继续深入对 RDB持久化的学习。在此之前，我们可以先思考一下如何实现一个持久化机制，毕竟这是很多中间件所需的一个模块。

首先，持久化保存的文件内容结构必须是紧凑的，特别对于数据库来说，需要持久化的数据量十分大，需要保证持久化文件不至于占用太多存储。
其次，进行持久化时，中间件应该还可以快速地响应用户请求，持久化的操作应该尽量少影响中间件的其他功能。
最后，毕竟持久化会消耗性能，如何在性能和数据安全性之间做出平衡，如何灵活配置触发持久化操作。

接下来我们将带着这些问题，到源码中寻求答案。

本文中的源码来自 Redis 4.0 ，RDB持久化过程的相关源码都在 rdb.c 文件中。其中大概的流程如下图所示。


![image.png](https://upload-images.jianshu.io/upload_images/623378-cc013bacdf6daf20.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图表明了三种触发 RDB 持久化的手段之间的整体关系。通过 `serverCron` 自动触发的 RDB 相当于直接调用了 bgsave 指令的流程进行处理。而 bgsave 的处理流程启动子进程后，调用了 save 指令的处理流程。

下面我们从 `serverCron` 自动触发逻辑开始研究。
 
### 自动触发 RDB 持久化

![](https://upload-images.jianshu.io/upload_images/623378-ed2f8d6095a0976f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图所示，`redisServer` 结构体的`save_params`指向拥有三个值的数组，该数组的值与 redis.conf 文件中 save 配置项一一对应。分别是 `save 900 1`、`save 300 10` 和 `save 60 10000`。`dirty` 记录着有多少键值发生变化，`lastsave`记录着上次 RDB 持久化的时间。

而 `serverCron` 函数就是遍历该数组的值，检查当前 Redis 状态是否符合触发 RDB 持久化的条件，比如说距离上次 RDB 持久化过去了 900 秒并且有至少一条数据发生变更。

```
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ....
    /* Check if a background saving or AOF rewrite in progress terminated. */
    /* 判断后台是否正在进行 rdb 或者 aof 操作 */
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {
        ....
    } else {
        // 到这儿就能确定 当前木有进行 rdb 或者 aof 操作
        // 遍历每一个 rdb 保存条件
         for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;

            //如果数据保存记录 大于规定的修改次数 且距离 上一次保存的时间大于规定时间或者上次BGSAVE命令执行成功，才执行 BGSAVE 操作
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK))
            {
                //记录日志
                serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                rdbSaveInfo rsi, *rsiptr;
                rsiptr = rdbPopulateSaveInfo(&rsi);
                // 异步保存操作
                rdbSaveBackground(server.rdb_filename,rsiptr);
                break;
            }
         }
    }
    ....
    server.cronloops++;
    return 1000/server.hz;
}
```

如果符合触发 RDB 持久化的条件，`serverCron`会调用`rdbSaveBackground`函数，也就是 bgsave 指令会触发的函数。

### 子进程后台执行 RDB 持久化

执行 bgsave 指令时，Redis 会先触发 `bgsaveCommand` 进行当前状态检查，然后才会调用`rdbSaveBackground`，其中的逻辑如下图所示。

![示意图](https://upload-images.jianshu.io/upload_images/623378-ca8e0411371013b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


`rdbSaveBackground` 函数中最主要的工作就是调用 `fork` 命令生成子流程，然后在子流程中执行 `rdbSave`函数，也就是 save 指令最终会触发的函数。
```
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi) {
    pid_t childpid;
    long long start;
    // 检查后台是否正在执行 aof 或者 rdb 操作
    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;
    // 拿出 数据保存记录，保存为 上次记录
    server.dirty_before_bgsave = server.dirty;
    // bgsave 时间
    server.lastbgsave_try = time(NULL);
    start = ustime();
    // fork 子进程
    if ((childpid = fork()) == 0) {
        int retval;
        /* 关闭子进程继承的 socket 监听 */
        closeListeningSockets(0);
        // 子进程 title 修改
        redisSetProcTitle("redis-rdb-bgsave");
        // 执行rdb 写入操作
        retval = rdbSave(filename,rsi);
        // 执行完毕以后
        ....
        // 退出子进程
        exitFromChild((retval == C_OK) ? 0 : 1);
    } else {
        /* 父进程，进行fork时间的统计和信息记录，比如说rdb_save_time_start、rdb_child_pid、和rdb_child_type */
        ....
        // rdb 保存开始时间 bgsave 子进程
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        updateDictResizePolicy();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```

为什么 Redis 使用子进程而不是线程来进行后台 RDB 持久化呢？主要是出于Redis性能的考虑，我们知道Redis对客户端响应请求的工作模型是单进程和单线程的，如果在主进程内启动一个线程，这样会造成对数据的竞争条件。所以为了避免使用锁降低性能，Redis选择启动新的子进程，独立拥有一份父进程的内存拷贝，以此为基础执行RDB持久化。

但是需要注意的是，fork 会消耗一定时间，并且父子进程所占据的内存是相同的，当 Redis 键值较大时，fork 的时间会很长，这段时间内 Redis 是无法响应其他命令的。除此之外，Redis 占据的内存空间会翻倍。


### 生成 RDB 文件，并且持久化到硬盘

Redis 的 `rdbSave` 函数是真正进行 RDB 持久化的函数，它的大致流程如下：

- 首先打开一个临时文件，
- 调用 `rdbSaveRio`函数，将当前 Redis 的内存信息写入到这个临时文件中，
- 接着调用 `fflush`、`fsync` 和 `fclose` 接口将文件写入磁盘中，
- 使用 `rename` 将临时文件改名为 正式的 RDB 文件，
- 最后记录 `dirty` 和 `lastsave `等状态信息。这些状态信息在 `serverCron`时会使用到。 

```
int rdbSave(char *filename, rdbSaveInfo *rsi) {
    char tmpfile[256];
    // 当前工作目录
    char cwd[MAXPATHLEN];
    FILE *fp;
    rio rdb;
    int error = 0;

    /* 生成tmpfile文件名 temp-[pid].rdb */
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    /* 打开文件 */
    fp = fopen(tmpfile,"w");
    .....
    /* 初始化rio结构 */
    rioInitWithFile(&rdb,fp);

    if (rdbSaveRio(&rdb,&error,RDB_SAVE_NONE,rsi) == C_ERR) {
        errno = error;
        goto werr;
    }

    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    /* 重新命名 rdb 文件，把之前临时的名称修改为正式的 rdb 文件名称 */
    if (rename(tmpfile,filename) == -1) {
        // 异常处理
        ....
    }
    // 写入完成，打印日志
    serverLog(LL_NOTICE,"DB saved on disk");
    // 清理数据保存记录
    server.dirty = 0;
    // 最后一次完成 SAVE 命令的时间
    server.lastsave = time(NULL);
    // 最后一次 bgsave 的状态置位 成功
    server.lastbgsave_status = C_OK;
    return C_OK;
    ....
}
```

这里要简单说一下 `fflush`和`fsync`的区别。它们俩都是用于刷缓存，但是所属的层次不同。`fflush`函数用于 `FILE*` 指针上，将缓存数据从应用层缓存刷新到内核中，而`fsync` 函数则更加底层，作用于文件描述符，用于将内核缓存刷新到物理设备上。

关于 Linux IO 的具体原理可以参考[《聊聊Linux IO》](https://mp.weixin.qq.com/s/3mKxTH2pfXFpDvvJnDtgEQ)

#### 内存数据到 RDB 文件

`rdbSaveRio` 会将 Redis 内存中的数据以相对紧凑的格式写入到文件中，其文件格式的示意图如下所示。

![](https://upload-images.jianshu.io/upload_images/623378-ee4aa3f1cdfd5f65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`rdbSaveRio`函数的写入大致流程如下：

- 先写入 REDIS 魔法值，然后是 RDB 文件的版本( rdb_version )，额外辅助信息 ( aux )。辅助信息中包含了 Redis 的版本，内存占用和复制库( repl-id )和偏移量( repl-offset )等。

- 然后 `rdbSaveRio` 会遍历当前 Redis 的所有数据库，将数据库的信息依次写入。先写入 `RDB_OPCODE_SELECTDB`识别码和数据库编号，接着写入`RDB_OPCODE_RESIZEDB`识别码和数据库键值数量和待失效键值数量，最后会遍历所有的键值，依次写入。

- 在写入键值时，当该键值有失效时间时，会先写入`RDB_OPCODE_EXPIRETIME_MS`识别码和失效时间，然后写入键值类型的识别码，最后再写入键和值。

- 写完数据库信息后，还会把 Lua 相关的信息写入，最后再写入 `RDB_OPCODE_EOF`结束符识别码和校验值。
```
int rdbSaveRio(rio *rdb, int *error, int flags, rdbSaveInfo *rsi) {
    snprintf(magic,sizeof(magic),"REDIS%04d",RDB_VERSION);
    /* 1 写入 magic字符'REDIS' 和 RDB 版本 */
    if (rdbWriteRaw(rdb,magic,9) == -1) goto werr;
    /* 2 写入辅助信息  REDIS版本,服务器操作系统位数,当前时间,复制信息比如repl-stream-db,repl-id和repl-offset等等数据*/
    if (rdbSaveInfoAuxFields(rdb,flags,rsi) == -1) goto werr;
    /* 3 遍历每一个数据库，逐个数据库数据保存 */
    for (j = 0; j < server.dbnum; j++) {
        /* 获取数据库指针地址和数据库字典 */
        redisDb *db = server.db+j;
        dict *d = db->dict;
        /* 3.1 写入数据库部分的开始标识 */
        if (rdbSaveType(rdb,RDB_OPCODE_SELECTDB) == -1) goto werr;
        /* 3.2 写入当前数据库号 */
        if (rdbSaveLen(rdb,j) == -1) goto werr;

        uint32_t db_size, expires_size;
        /* 获取数据库字典大小和过期键字典大小 */
        db_size = (dictSize(db->dict) <= UINT32_MAX) ?
                                dictSize(db->dict) :
                                UINT32_MAX;
        expires_size = (dictSize(db->expires) <= UINT32_MAX) ?
                                dictSize(db->expires) :
                                UINT32_MAX;
        /* 3.3 写入当前待写入数据的类型，此处为 RDB_OPCODE_RESIZEDB，表示数据库大小 */
        if (rdbSaveType(rdb,RDB_OPCODE_RESIZEDB) == -1) goto werr;
        /* 3.4 写入获取数据库字典大小和过期键字典大小 */
        if (rdbSaveLen(rdb,db_size) == -1) goto werr;
        if (rdbSaveLen(rdb,expires_size) == -1) goto werr;
        /* 4 遍历当前数据库的键值对 */
        while((de = dictNext(di)) != NULL) {
            sds keystr = dictGetKey(de);
            robj key, *o = dictGetVal(de);
            long long expire;

            /* 初始化 key，因为操作的是 key 字符串对象，而不是直接操作 键的字符串内容 */
            initStaticStringObject(key,keystr);
            /* 获取键的过期数据 */
            expire = getExpire(db,&key);
            /* 4.1 保存键值对数据 */
            if (rdbSaveKeyValuePair(rdb,&key,o,expire) == -1) goto werr;
        }

    }

    /* 5 保存 Lua 脚本*/
    if (rsi && dictSize(server.lua_scripts)) {
        di = dictGetIterator(server.lua_scripts);
        while((de = dictNext(di)) != NULL) {
            robj *body = dictGetVal(de);
            if (rdbSaveAuxField(rdb,"lua",3,body->ptr,sdslen(body->ptr)) == -1)
                goto werr;
        }
        dictReleaseIterator(di);
    }

    /* 6 写入结束符 */
    if (rdbSaveType(rdb,RDB_OPCODE_EOF) == -1) goto werr;

    /* 7 写入CRC64校验和 */
    cksum = rdb->cksum;
    memrev64ifbe(&cksum);
    if (rioWrite(rdb,&cksum,8) == 0) goto werr;
    return C_OK;
}
```

`rdbSaveRio `在写键值时，会调用`rdbSaveKeyValuePair` 函数。该函数会依次写入键值的过期时间，键的类型，键和值。

```
int rdbSaveKeyValuePair(rio *rdb, robj *key, robj *val, long long expiretime)
{
    /* 如果有过期信息 */
    if (expiretime != -1) {
        /* 保存过期信息标识 */
        if (rdbSaveType(rdb,RDB_OPCODE_EXPIRETIME_MS) == -1) return -1;
        /* 保存过期具体数据内容 */
        if (rdbSaveMillisecondTime(rdb,expiretime) == -1) return -1;
    }

    /* Save type, key, value */
    /* 保存键值对 类型的标识 */
    if (rdbSaveObjectType(rdb,val) == -1) return -1;
    /* 保存键值对 键的内容 */
    if (rdbSaveStringObject(rdb,key) == -1) return -1;
    /* 保存键值对 值的内容 */
    if (rdbSaveObject(rdb,val) == -1) return -1;
    return 1;
}
```
根据键的不同类型写入不同格式，各种键值的类型和格式如下所示。

![](https://upload-images.jianshu.io/upload_images/623378-9337bcd51d556022.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Redis 有庞大的对象和数据结构体系，它使用六种底层数据结构构建了包含字符串对象、列表对象、哈希对象、集合对象和有序集合对象的对象系统。感兴趣的同学可以参考 [《十二张图带你了解 Redis 的数据结构和对象系统》](https://mp.weixin.qq.com/s/gQnuynv6XPD_aeIBQBeI2Q)一文。

不同的数据结构进行 RDB 持久化的格式都不同。我们今天只看一下集合对象是如何持久化的。
```
ssize_t rdbSaveObject(rio *rdb, robj *o) {
    ssize_t n = 0, nwritten = 0;
    ....
    } else if (o->type == OBJ_SET) {
        /* Save a set value */
        if (o->encoding == OBJ_ENCODING_HT) {
            dict *set = o->ptr;
            // 集合迭代器
            dictIterator *di = dictGetIterator(set);
            dictEntry *de;
            // 写入集合长度
            if ((n = rdbSaveLen(rdb,dictSize(set))) == -1) return -1;
            nwritten += n;
            // 遍历集合元素
            while((de = dictNext(di)) != NULL) {
                sds ele = dictGetKey(de);
                // 以字符串的形式写入，因为是SET 所以只写入 Key 即可
                if ((n = rdbSaveRawString(rdb,(unsigned char*)ele,sdslen(ele)))
                    == -1) return -1;
                nwritten += n;
            }
            dictReleaseIterator(di);
        } 
    .....
    return nwritten;
}
```

![](https://upload-images.jianshu.io/upload_images/623378-289642734e6eb81b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


