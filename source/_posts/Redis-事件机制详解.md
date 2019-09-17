---
title: Redis 事件机制详解
tags: redis
abbrlink: 1aa2da89
date: 2019-08-04 18:37:29
---
Redis 采用事件驱动机制来处理大量的网络IO。它并没有使用 libevent 或者 libev 这样的成熟开源方案，而是自己实现一个非常简洁的事件驱动库 ae_event。

Redis中的事件驱动库只关注网络IO，以及定时器。该事件库处理下面两类事件：

- 文件事件(file  event)：用于处理 Redis 服务器和客户端之间的网络IO。
- 时间事件(time  eveat)：Redis 服务器中的一些操作（比如serverCron函数）需要在给定的时间点执行，而时间事件就是处理这类定时操作的。


事件驱动库的代码主要是在src/ae.c中实现的，其示意图如下所示。

![事件管理器示意图](/images/19_85/image1.png)

`aeEventLoop`是整个事件驱动的核心，它管理着文件事件表和时间事件列表，
不断地循环处理着就绪的文件事件和到期的时间事件。下面我们就先分别介绍文件事件和时间事件，然后讲述相关的`aeEventLoop`源码实现。

### 文件事件

Redis基于Reactor模式开发了自己的网络事件处理器，也就是文件事件处理器。文件事件处理器使用IO多路复用技术，同时监听多个套接字，并为套接字关联不同的事件处理函数。当套接字的可读或者可写事件触发时，就会调用相应的事件处理函数。

Redis 使用的IO多路复用技术主要有：`select`、`epoll`、`evport`和`kqueue`等。每个IO多路复用函数库在 Redis 源码中都对应一个单独的文件，比如ae_select.c，ae_epoll.c， ae_kqueue.c等。Redis 会根据不同的操作系统，按照不同的优先级选择多路复用技术。事件响应框架一般都采用该架构，比如 netty 和 libevent。

![示意图](/images/19_85/image2.png)

如下图所示，文件事件处理器有四个组成部分，它们分别是套接字、I/O多路复用程序、文件事件分派器以及事件处理器。

![示意图](/images/19_85/image3.png)


文件事件是对套接字操作的抽象，每当一个套接字准备好执行 accept、read、write和 close 等操作时，就会产生一个文件事件。因为 Redis 通常会连接多个套接字，所以多个文件事件有可能并发的出现。

I/O多路复用程序负责监听多个套接字，并向文件事件派发器传递那些产生了事件的套接字。

尽管多个文件事件可能会并发地出现，但I/O多路复用程序总是会将所有产生的套接字都放到同一个队列(也就是后文中描述的`aeEventLoop`的`fired`就绪事件表)里边，然后文件事件处理器会以有序、同步、单个套接字的方式处理该队列中的套接字，也就是处理就绪的文件事件。

![一次请求的过程示意图](/images/19_85/image4.png)

所以，一次 Redis 客户端与服务器进行连接并且发送命令的过程如上图所示。

- 客户端向服务端发起建立 socket 连接的请求，那么监听套接字将产生 AE_READABLE 事件，触发**连接应答处理器**执行。处理器会对客户端的连接请求进行应答，然后创建客户端套接字，以及客户端状态，并将客户端套接字的 AE_READABLE 事件与**命令请求处理器**关联。
- 客户端建立连接后，向服务器发送命令，那么客户端套接字将产生 AE_READABLE 事件，触发**命令请求处理器**执行，处理器读取客户端命令，然后传递给相关程序去执行。
- 执行命令获得相应的命令回复，为了将命令回复传递给客户端，服务器将客户端套接字的 AE_WRITEABLE 事件与**命令回复处理器**关联。当客户端试图读取命令回复时，客户端套接字产生 AE_WRITEABLE 事件，触发命令**回复处理器**将命令回复全部写入到套接字中。

### 时间事件

Redis 的时间事件分为以下两类：
- 定时事件：让一段程序在指定的时间之后执行一次。
- 周期性事件：让一段程序每隔指定时间就执行一次。

Redis 的时间事件的具体定义结构如下所示。

```
typedef struct aeTimeEvent {
    /* 全局唯一ID */
    long long id; /* time event identifier. */
    /* 秒精确的UNIX时间戳，记录时间事件到达的时间*/
    long when_sec; /* seconds */
    /* 毫秒精确的UNIX时间戳，记录时间事件到达的时间*/
    long when_ms; /* milliseconds */
    /* 时间处理器 */
    aeTimeProc *timeProc;
    /* 事件结束回调函数，析构一些资源*/
    aeEventFinalizerProc *finalizerProc;
    /* 私有数据 */
    void *clientData;
    /* 前驱节点 */
    struct aeTimeEvent *prev;
    /* 后继节点 */
    struct aeTimeEvent *next;
} aeTimeEvent;
```

一个时间事件是定时事件还是周期性事件取决于时间处理器的返回值：
- 如果返回值是 AE_NOMORE，那么这个事件是一个定时事件，该事件在达到后删除，之后不会再重复。
- 如果返回值是非 AE_NOMORE 的值，那么这个事件为周期性事件，当一个时间事件到达后，服务器会根据时间处理器的返回值，对时间事件的 `when` 属性进行更新，让这个事件在一段时间后再次达到。

Redis 将所有时间事件都放在一个无序链表中，每次 Redis 会遍历整个链表，查找所有已经到达的时间事件，并且调用相应的事件处理器。

介绍完文件事件和时间事件，我们接下来看一下 `aeEventLoop`的具体实现。
### 创建事件管理器

Redis 服务端在其初始化函数 `initServer`中，会创建事件管理器`aeEventLoop`对象。

函数`aeCreateEventLoop`将创建一个事件管理器，主要是初始化 `aeEventLoop`的各个属性值，比如`events`、`fired`、`timeEventHead`和`apidata`：
- 首先创建`aeEventLoop`对象。
- 初始化未就绪文件事件表、就绪文件事件表。`events`指针指向未就绪文件事件表、`fired`指针指向就绪文件事件表。表的内容在后面添加具体事件时进行初变更。
- 初始化时间事件列表，设置`timeEventHead`和`timeEventNextId`属性。
- 调用`aeApiCreate` 函数创建`epoll`实例，并初始化 `apidata`。

```
aeEventLoop *aeCreateEventLoop(int setsize) {
    aeEventLoop *eventLoop;
    int i;
    /* 创建事件状态结构 */
    if ((eventLoop = zmalloc(sizeof(*eventLoop))) == NULL) goto err;
    /* 创建未就绪事件表、就绪事件表 */
    eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
    eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);
    if (eventLoop->events == NULL || eventLoop->fired == NULL) goto err;
    /* 设置数组大小 */
    eventLoop->setsize = setsize;
    /* 初始化执行最近一次执行时间 */
    eventLoop->lastTime = time(NULL);
    /* 初始化时间事件结构 */
    eventLoop->timeEventHead = NULL;
    eventLoop->timeEventNextId = 0;
    eventLoop->stop = 0;
    eventLoop->maxfd = -1;
    eventLoop->beforesleep = NULL;
    eventLoop->aftersleep = NULL;
    /* 将多路复用io与事件管理器关联起来 */
    if (aeApiCreate(eventLoop) == -1) goto err;
    /* 初始化监听事件 */
    for (i = 0; i < setsize; i++)
        eventLoop->events[i].mask = AE_NONE;
    return eventLoop;
err:
   .....
}
```

`aeApiCreate` 函数首先创建了`aeApiState`对象，初始化了epoll就绪事件表；然后调用`epoll_create`创建了`epoll`实例，最后将该`aeApiState`赋值给`apidata`属性。

`aeApiState`对象中`epfd`存储`epoll`的标识，`events`是一个`epoll`就绪事件数组，当有`epoll`事件发生时，所有发生的`epoll`事件和其描述符将存储在这个数组中。这个就绪事件数组由应用层开辟空间、内核负责把所有发生的事件填充到该数组。

```
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    /* 初始化epoll就绪事件表 */
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    if (!state->events) {
        zfree(state);
        return -1;
    }
    /* 创建 epoll 实例 */
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    /* 事件管理器与epoll关联 */
    eventLoop->apidata = state;
    return 0;
}
typedef struct aeApiState {
    /* epoll_event 实例描述符*/
    int epfd;
    /* 存储epoll就绪事件表 */
    struct epoll_event *events;
} aeApiState;
```

### 创建文件事件

`aeFileEvent`是文件事件结构，对于每一个具体的事件，都有读处理函数和写处理函数等。Redis 调用`aeCreateFileEvent`函数针对不同的套接字的读写事件注册对应的文件事件。

```
typedef struct aeFileEvent {
    /* 监听事件类型掩码,值可以是 AE_READABLE 或 AE_WRITABLE */
    int mask;
    /* 读事件处理器 */
    aeFileProc *rfileProc;
    /* 写事件处理器 */
    aeFileProc *wfileProc;
    /* 多路复用库的私有数据 */
    void *clientData;
} aeFileEvent;
/* 使用typedef定义的处理器函数的函数类型 */
typedef void aeFileProc(struct aeEventLoop *eventLoop, 
int fd, void *clientData, int mask);

```

比如说，Redis 进行主从复制时，从服务器需要主服务器建立连接，它会发起一个 socekt连接，然后调用`aeCreateFileEvent`函数针对发起的socket的读写事件注册了对应的事件处理器，也就是`syncWithMaster`函数。

```
aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster,NULL);
/* 符合aeFileProc的函数定义 */
void syncWithMaster(aeEventLoop *el, int fd, void *privdata, int mask) {....}
```

`aeCreateFileEvent`的参数`fd`指的是具体的`socket`套接字，`proc`指`fd`产生事件时，具体的处理函数，`clientData`则是回调处理函数时需要传入的数据。
`aeCreateFileEvent`主要做了三件事情：
- 以`fd`为索引，在`events`未就绪事件表中找到对应事件。
- 调用`aeApiAddEvent`函数，该事件注册到具体的底层 I/O 多路复用中，本例为epoll。
- 填充事件的回调、参数、事件类型等参数。

```
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
                       aeFileProc *proc, void *clientData)
{
    /* 取出 fd 对应的文件事件结构, fd 代表具体的 socket 套接字 */
    aeFileEvent *fe = &eventLoop->events[fd];
    /* 监听指定 fd 的指定事件 */
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    /* 置文件事件类型，以及事件的处理器 */
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    /* 私有数据 */
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```
如上文所说，Redis 基于的底层 I/O 多路复用库有多套，所以`aeApiAddEvent`也有多套实现，下面的源码是`epoll`下的实现。其核心操作就是调用`epoll`的`epoll_ctl`函数来向`epoll`注册响应事件。有关`epoll`相关的知识可以看一下[《Java NIO源码分析》](https://mp.weixin.qq.com/s/O40MlDhAgCmcyUgu2jX1dg)
```
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    /* 如果 fd 没有关联任何事件，那么这是一个 ADD 操作。如果已经关联了某个/某些事件，那么这是一个 MOD 操作。 */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    /* 注册事件到 epoll */
    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    /* 调用epoll_ctl 系统调用，将事件加入epoll中 */
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```

### 事件处理

因为 Redis 中同时存在文件事件和时间事件两个事件类型，所以服务器必须对这两个事件进行调度，决定何时处理文件事件，何时处理时间事件，以及如何调度它们。

`aeMain`函数以一个无限循环不断地调用`aeProcessEvents`函数来处理所有的事件。

```
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        /* 如果有需要在事件处理前执行的函数，那么执行它 */
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        /* 开始处理事件*/
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

下面是`aeProcessEvents`的伪代码，它会首先计算距离当前时间最近的时间事件，以此计算一个超时时间；然后调用`aeApiPoll`函数去等待底层的I/O多路复用事件就绪；`aeApiPoll`函数返回之后，会处理所有已经产生文件事件和已经达到的时间事件。

```
/* 伪代码 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    /* 获取到达时间距离当前时间最接近的时间事件*/
    time_event = aeSearchNearestTimer();
    /* 计算最接近的时间事件距离到达还有多少毫秒*/
    remaind_ms = time_event.when - unix_ts_now();
    /* 如果事件已经到达，那么remaind_ms为负数，将其设置为0 */
    if (remaind_ms < 0) remaind_ms = 0;
    /* 根据 remaind_ms 的值，创建 timeval 结构*/
    timeval = create_timeval_with_ms(remaind_ms);
    /* 阻塞并等待文件事件产生，最大阻塞时间由传入的 timeval 结构决定，如果remaind_ms 的值为0，则aeApiPoll 调用后立刻返回，不阻塞*/
    /* aeApiPoll调用epoll_wait函数，等待I/O事件*/
    aeApiPoll(timeval);
    /* 处理所有已经产生的文件事件*/
    processFileEvents();
    /* 处理所有已经到达的时间事件*/
    processTimeEvents();
}
```

与`aeApiAddEvent`类似，`aeApiPoll`也有多套实现，它其实就做了两件事情，调用`epoll_wait`阻塞等待`epoll`的事件就绪，超时时间就是之前根据最快达到时间事件计算而来的超时时间；然后将就绪的`epoll`事件转换到fired就绪事件。`aeApiPoll`就是上文所说的I/O多路复用程序。具体过程如下图所示。

![aeApiPoll示意图](/images/19_85/image5.png)


```
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) 
{
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;
    // 调用epoll_wait函数，等待时间为最近达到时间事件的时间计算而来。
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    // 有至少一个事件就绪？
    if (retval > 0) 
    {
        int j;
        /*为已就绪事件设置相应的模式，并加入到 eventLoop 的 fired 数组中*/
        numevents = retval;
        for (j = 0; j < numevents; j++) 
	{
            int mask = 0;
            struct epoll_event *e = state->events+j;
            if (e->events & EPOLLIN)
		mask |= AE_READABLE;
            if (e->events & EPOLLOUT)
		mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) 
		mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP)
		mask |= AE_WRITABLE;
            /* 设置就绪事件表元素 */
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    
    // 返回已就绪事件个数
    return numevents;
}
```

`processFileEvent`是处理就绪文件事件的伪代码，也是上文所述的文件事件分派器，它其实就是遍历`fired`就绪事件表，然后根据对应的事件类型来调用事件中注册的不同处理器，读事件调用`rfileProc`，而写事件调用`wfileProc`。

```
void processFileEvent(int numevents) {
    for (j = 0; j < numevents; j++) {
            /* 从已就绪数组中获取事件 */
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0;
            int invert = fe->mask & AE_BARRIER;
	        /* 读事件 */
            if (!invert && fe->mask & mask & AE_READABLE) {
                /* 调用读处理函数 */
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
            /* 写事件. */
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }
            if (invert && fe->mask & mask & AE_READABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }
            processed++;
        }
    }
}
```
而`processTimeEvents`是处理时间事件的函数，它会遍历`aeEventLoop`的事件事件列表，如果时间事件到达就执行其`timeProc`函数，并根据函数的返回值是否等于`AE_NOMORE`来决定该时间事件是否是周期性事件，并修改器到达时间。
```
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;
    time_t now = time(NULL);
    ....
    eventLoop->lastTime = now;

    te = eventLoop->timeEventHead;
    maxId = eventLoop->timeEventNextId-1;
    /* 遍历时间事件链表 */
    while(te) {
        long now_sec, now_ms;
        long long id;

        /* 删除需要删除的时间事件 */
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }

        /* id 大于最大maxId,是该循环周期生成的时间事件，不处理 */
        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        aeGetTime(&now_sec, &now_ms);
        /* 事件已经到达，调用其timeProc函数*/
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;

            id = te->id;
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            /* 如果返回值不等于 AE_NOMORE,表示是一个周期性事件，修改其when_sec和when_ms属性*/
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
                /* 一次性事件，标记为需删除，下次遍历时会删除*/
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
    return processed;
}
```

### 删除事件
当不在需要某个事件时，需要把事件删除掉。例如: 如果fd同时监听读事件、写事件。当不在需要监听写事件时，可以把该fd的写事件删除。

`aeDeleteEventLoop`函数的执行过程总结为以下几个步骤
1、根据`fd`在未就绪表中查找到事件
2、取消该`fd`对应的相应事件标识符
3、调用`aeApiFree`函数，内核会将epoll监听红黑树上的相应事件监听取消。

### 后记

接下来，我们会继续学习 Redis 的主从复制相关的原理，欢迎大家持续关注。

![](/images/logo.png)
