---
title: 详解 Redis 内存管理机制和实现
tags: redis
category: Redis
abbrlink: e66f8da0
date: 2019-10-26 22:46:39
---
Redis是一个基于内存的键值数据库，其内存管理是非常重要的。本文内存管理的内容包括：过期键的懒性删除和过期删除以及内存溢出控制策略。


### 最大内存限制

Redis使用 maxmemory 参数限制最大可用内存，默认值为0，表示无限制。限制内存的目的主要 有： 
- 用于缓存场景，当超出内存上限 maxmemory 时使用 LRU 等删除策略释放空间。 
- 防止所用内存超过服务器物理内存。因为 Redis 默认情况下是会尽可能多使用服务器的内存，可能会出现服务器内存不足，导致 Redis 进程被杀死。

![](/images/19_1026/image1.webp)

maxmemory 限制的是Redis实际使用的内存量，也就是 used_memory统计项对应的内存。由于内存碎片率的存在，实际消耗的内存 可能会比maxmemory设置的更大，实际使用时要小心这部分内存溢出。具体Redis 内存监控的内容请查看[一文了解 Redis 内存监控和内存消耗](https://mp.weixin.qq.com/s/SrQIGL_X8wC1eFsGu8gBXg)。

Redis默认无限使用服务器内存，为防止极端情况下导致系统内存耗 尽，建议所有的Redis进程都要配置maxmemory。 在保证物理内存可用的情况下，系统中所有Redis实例可以调整 maxmemory参数来达到自由伸缩内存的目的。

### 内存回收策略

Redis 回收内存大致有两个机制：一是删除到达过期时间的键值对象；二是当内存达到 maxmemory 时触发内存移除控制策略，强制删除选择出来的键值对象。

#### 删除过期键对象 

Redis 所有的键都可以设置过期属性，内部保存在过期表中，键值表和过期表的结果如下图所示。当 Redis保存大量的键，对每个键都进行精准的过期删除可能会导致消耗大量的 CPU，会阻塞 Redis 的主线程，拖累 Redis 的性能，因此 Redis 采用惰性删除和定时任务删除机制实现过期键的内存回收。

![](/images/19_1026/image2.webp)


惰性删除是指当客户端操作带有超时属性的键时，会检查是否超过键的过期时间，然后会同步或者异步执行删除操作并返回键已经过期。这样可以节省 CPU成本考虑，不需要单独维护过期时间链表来处理过期键的删除。

过期键的惰性删除策略由 db.c/expireifNeeded 函数实现，所有对数据库的读写命令执行之前都会调用 expireifNeeded 来检查命令执行的键是否过期。如果键过期，expireifNeeded 会将过期键从键值表和过期表中删除，然后同步或者异步释放对应对象的空间。源码展示的时 Redis 4.0 版本。

expireIfNeeded 先从过期表中获取键对应的过期时间，如果当前时间已经超过了过期时间(lua脚本执行则有特殊逻辑，详看代码注释)，则进入删除键流程。删除键流程主要进行了三件事：
- 一是删除操作命令传播，通知 slave 实例并存储到 AOF 缓冲区中
- 二是记录键空间事件，
- 三是根据 lazyfree_lazy_expire 是否开启进行异步删除或者异步删除操作。 

```
int expireIfNeeded(redisDb *db, robj *key) {
    // 获取键的过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;
    // 键没有过期时间
    if (when < 0) return 0;
    // 实例正在从硬盘 laod 数据，比如说 RDB 或者 AOF
    if (server.loading) return 0;

    // 当执行lua脚本时，只有键在lua一开始执行时
    // 就到了过期时间才算过期，否则在lua执行过程中不算失效
    now = server.lua_caller ? server.lua_time_start : mstime();

    // 当本实例是slave时，过期键的删除由master发送过来的
    // del 指令控制。但是这个函数还是将正确的信息返回给调用者。
    if (server.masterhost != NULL) return now > when;
    // 判断是否未过期
    if (now <= when) return 0;

    // 代码到这里，说明键已经过期，而且需要被删除
    server.stat_expiredkeys++;
    // 命令传播，到 slave 和 AOF
    propagateExpire(db,key,server.lazyfree_lazy_expire);
    // 键空间通知使得客户端可以通过订阅频道或模式， 来接收那些以某种方式改动了 Redis 数据集的事件。
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    // 如果是惰性删除，调用dbAsyncDelete，否则调用 dbSyncDelete
    return server.lazyfree_lazy_expire ? dbAsyncDelete(db,key) :
                                         dbSyncDelete(db,key);
}
```

![](/images/19_1026/image3.webp)


上图是写命令传播的示意图，删除命令的传播和它一致。propagateExpire 函数先调用 feedAppendOnlyFile 函数将命令同步到 AOF 的缓冲区中，然后调用 replicationFeedSlaves函数将命令同步到所有的 slave 中。Redis 复制的机制可以查看[Redis 复制过程详解](https://mp.weixin.qq.com/s/0VVYTyAI1egfs2Fxcrme3A)。

```
// 将命令传递到slave和AOF缓冲区。maser删除一个过期键时会发送Del命令到所有的slave和AOF缓冲区
void propagateExpire(redisDb *db, robj *key, int lazy) {
    robj *argv[2];
    // 生成同步的数据
    argv[0] = lazy ? shared.unlink : shared.del;
    argv[1] = key;
    incrRefCount(argv[0]);
    incrRefCount(argv[1]);
    // 如果开启了 AOF 则追加到 AOF 缓冲区中
    if (server.aof_state != AOF_OFF)
        feedAppendOnlyFile(server.delCommand,db->id,argv,2);
    // 同步到所有 slave
    replicationFeedSlaves(server.slaves,db->id,argv,2);

    decrRefCount(argv[0]);
    decrRefCount(argv[1]);
}
```

dbAsyncDelete 函数会先调用 dictDelete 来删除过期表中的键，然后处理键值表中的键值对象。它会根据值的占用的空间来选择是直接释放值对象，还是交给 bio 异步释放值对象。判断依据就是值的估计大小是否大于 LAZYFREE_THRESHOLD 阈值。键对象和 dictEntry 对象则都是直接被释放。

![](/images/19_1026/image4.webp)


```
#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {
    // 删除该键在过期表中对应的entry
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

    // unlink 该键在键值表对应的entry
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    // 如果该键值占用空间非常小，懒删除反而效率低。所以只有在一定条件下，才会异步删除
    if (de) {
        robj *val = dictGetVal(de);
        size_t free_effort = lazyfreeGetFreeEffort(val);
        // 如果释放这个对象消耗很多，并且值未被共享(refcount == 1)则将其加入到懒删除列表
        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1);
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
            dictSetVal(db->dict,de,NULL);
        }
    }

    // 释放键值对，或者只释放key，而将val设置为NULL来后续懒删除
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        // slot 和 key 的映射关系是用于快速定位某个key在哪个 slot中。
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```

dictUnlink 会将键值从键值表中删除，但是却不释放 key、val和对应的表entry对象，而是将其直接返回，然后再调用dictFreeUnlinkedEntry进行释放。dictDelete 是它的兄弟函数，但是会直接释放相应的对象。二者底层都通过调用 dictGenericDelete来实现。dbAsyncDelete d的兄弟函数 dbSyncDelete 就是直接调用dictDelete来删除过期键。

```
void dictFreeUnlinkedEntry(dict *d, dictEntry *he) {
    if (he == NULL) return;
    // 释放key对象
    dictFreeKey(d, he);
    // 释放值对象，如果它不为null
    dictFreeVal(d, he);
    // 释放 dictEntry 对象
    zfree(he);
}
```

Redis 有自己的 bio 机制，主要是处理 AOF 落盘、懒删除逻辑和关闭大文件fd。bioCreateBackgroundJob 函数将释放值对象的 job 加入到队列中，bioProcessBackgroundJobs会从队列中取出任务，根据类型进行对应的操作。

```
void *bioProcessBackgroundJobs(void *arg) {
    .....
    while(1) {
        listNode *ln;

        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        if (type == BIO_CLOSE_FILE) {
            close((long)job->arg1);
        } else if (type == BIO_AOF_FSYNC) {
            aof_fsync((long)job->arg1);
        } else if (type == BIO_LAZY_FREE) {
            // 根据参数来决定要做什么。有参数1则要释放它，有参数2和3是释放两个键值表
            // 过期表，也就是释放db 只有参数三是释放跳表
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1);
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);
        }
        zfree(job);
        ......
    }
}
```

dbSyncDelete 则是直接删除过期键，并且将键、值和 DictEntry 对象都释放。

```
int dbSyncDelete(redisDb *db, robj *key) {
    // 删除过期表中的entry
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    // 删除键值表中的entry
    if (dictDelete(db->dict,key->ptr) == DICT_OK) {
        // 如果开启了集群，则删除slot 和 key 映射表中key记录。
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```



但是单独用这种方式存在内存泄露的问题，当过期键一直没有访问将无法得到及时删除，从而导致内存不能及时释放。正因为如此，Redis还提供另一种定时任 务删除机制作为惰性删除的补充。

Redis 内部维护一个定时任务，默认每秒运行10次（通过配置控制）。定时任务中删除过期键逻辑采用了自适应算法，根据键的 过期比例、使用快慢两种速率模式回收键，流程如下图所示。

![](/images/19_1026/image5.webp)


- 1）定时任务首先根据快慢模式( 慢模型扫描的键的数量以及可以执行时间都比快模式要多 )和相关阈值配置计算计算本周期最大执行时间、要检查的数据库数量以及每个数据库扫描的键数量。
- 2)  从上次定时任务未扫描的数据库开始，依次遍历各个数据库。
- 3）从数据库中随机选手 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP 个键，如果发现是过期键，则调用 activeExpireCycleTryExpire 函数删除它。
- 4）如果执行时间超过了设定的最大执行时间，则退出，并设置下一次使用慢模式执行。
- 5）未超时的话，则判断是否采样的键中是否有25%的键是过期的，如果是则继续扫描当前数据库，跳到第3步。否则开始扫描下一个数据库。

定期删除策略由 expire.c/activeExpireCycle 函数实现。在redis事件驱动的循环中的eventLoop->beforesleep和
周期性操作 databasesCron 都会调用 activeExpireCycle 来处理过期键。但是二者传入的 type 值不同，一个是ACTIVE_EXPIRE_CYCLE_SLOW 另外一个是ACTIVE_EXPIRE_CYCLE_FAST。activeExpireCycle 在规定的时间，分多次遍历各个数据库，从 expires 字典中随机检查一部分过期键的过期时间，删除其中的过期键，相关源码如下所示。

```
void activeExpireCycle(int type) {
    // 上次检查的db
    static unsigned int current_db = 0; 
    // 上次检查的最大执行时间
    static int timelimit_exit = 0;
    // 上一次快速模式运行时间
    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

    int j, iteration = 0;
    // 每次检查周期要遍历的DB数
    int dbs_per_call = CRON_DBS_PER_CALL;
    long long start = ustime(), timelimit, elapsed;

    ..... // 一些状态时不进行检查，直接返回

    // 如果上次周期因为执行达到了最大执行时间而退出，则本次遍历所有db,否则遍历db数等于 CRON_DBS_PER_CALL
    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    // 根据ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC计算本次最大执行时间
    timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;
    // 如果是快速模式，则最大执行时间为ACTIVE_EXPIRE_CYCLE_FAST_DURATION
    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION; /* in microseconds. */
    // 采样记录
    long total_sampled = 0;
    long total_expired = 0;
    // 依次遍历 dbs_per_call 个 db
    for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
        int expired;
        redisDb *db = server.db+(current_db % server.dbnum);
        // 将db数增加，一遍下一次继续从这个db开始遍历
        current_db++;

        do {
            ..... // 申明变量和一些情况下 break
            if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
                num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;
            // 主要循环，在过期表中进行随机采样，判断是否比率大于25%
            while (num--) {
                dictEntry *de;
                long long ttl;

                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                ttl = dictGetSignedIntegerVal(de)-now;
                // 删除过期键
                if (activeExpireCycleTryExpire(db,de,now)) expired++;
                if (ttl > 0) {
                    /* We want the average TTL of keys yet not expired. */
                    ttl_sum += ttl;
                    ttl_samples++;
                }
                total_sampled++;
            }
            // 记录过期总数
            total_expired += expired;
            // 即使有很多键要过期，也不阻塞很久，如果执行超过了最大执行时间，则返回
            if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
                elapsed = ustime()-start;
                if (elapsed > timelimit) {
                    timelimit_exit = 1;
                    server.stat_expired_time_cap_reached_count++;
                    break;
                }
            }
            // 当比率小于25%时返回
        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
    }
    .....// 更新一些server的记录数据
}

```
activeExpireCycleTryExpire 函数的实现就和 expireIfNeeded 类似，这里就不赘述了。

```
int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now) {
    long long t = dictGetSignedIntegerVal(de);
    if (now > t) {
        sds key = dictGetKey(de);
        robj *keyobj = createStringObject(key,sdslen(key));

        propagateExpire(db,keyobj,server.lazyfree_lazy_expire);
        if (server.lazyfree_lazy_expire)
            dbAsyncDelete(db,keyobj);
        else
            dbSyncDelete(db,keyobj);
        notifyKeyspaceEvent(NOTIFY_EXPIRED,
            "expired",keyobj,db->id);
        decrRefCount(keyobj);
        server.stat_expiredkeys++;
        return 1;
    } else {
        return 0;
    }
}
```
定期删除策略的关键点就是删除操作执行的时长和频率：
- 如果删除操作太过频繁或者执行时间太长，就对 CPU 时间不是很友好，CPU 时间过多的消耗在删除过期键上。
- 如果删除操作执行太少或者执行时间太短，就不能及时删除过期键，导致内存浪费。

#### 内存溢出控制策略

当Redis所用内存达到maxmemory上限时会触发相应的溢出控制策略。 具体策略受maxmemory-policy参数控制，Redis支持6种策略，如下所示： 
- 1）noeviction：默认策略，不会删除任何数据，拒绝所有写入操作并返 回客户端错误信息（error）OOM command not allowed when used memory，此 时Redis只响应读操作。 
- 2）volatile-lru：根据LRU算法删除设置了超时属性（expire）的键，直 到腾出足够空间为止。如果没有可删除的键对象，回退到noeviction策略。 
- 3）allkeys-lru：根据LRU算法删除键，不管数据有没有设置超时属性， 直到腾出足够空间为止。 
- 4）allkeys-random：随机删除所有键，直到腾出足够空间为止。 
- 5）volatile-random：随机删除过期键，直到腾出足够空间为止。 
- 6）volatile-ttl：根据键值对象的ttl属性，删除最近将要过期数据。如果没有，回退到noeviction策略。


内存溢出控制策略可以使用 config set maxmemory-policy {policy} 语句进行动态配置。Redis 提供了丰富的空间溢出控制策略，我们可以根据自身业务需要进行选择。

当设置 volatile-lru 策略时，保证具有过期属性的键可以根据 LRU 剔除，而未设置超时的键可以永久保留。还可以采用allkeys-lru 策略把 Redis 变为纯缓存服务器使用。

当Redis因为内存溢出删除键时，可以通过执行 info stats 命令查看 evicted_keys 指标找出当前 Redis 服务器已剔除的键数量。 

每次Redis执行命令时如果设置了maxmemory参数，都会尝试执行回收 内存操作。当Redis一直工作在内存溢出（used_memory>maxmemory）的状态下且设置非 noeviction 策略时，会频繁地触发回收内存的操作，影响Redis 服务器的性能，这一点千万要引起注意。

![](/images/logo.png)
