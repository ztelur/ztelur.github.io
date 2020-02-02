---
title: 基于Redis和Lua的分布式限流
tags: 限流
categories: 
 - Redis
abbrlink: ad1b673b
date: 2019-04-06 10:51:31
---


&emsp;Java单机限流可以使用AtomicInteger，RateLimiter或Semaphore来实现，但是上述方案都不支持集群限流。集群限流的应用场景有两个，一个是网关，常用的方案有Nginx限流和Spring Cloud Gateway，另一个场景是与外部或者下游服务接口的交互，因为接口限制必须进行限流。

&emsp;本文的主要内容为：
- Redis和Lua的使用场景和注意事项，特别是KEY映射的问题
- Spring Cloud Gateway中限流的实现

### 集群限流的难点
&emsp;在上篇Guava RateLimiter的分析[文章](https://mp.weixin.qq.com/s?__biz=MzU2MDYwMDMzNQ==&mid=2247483781&idx=1&sn=05a2de89cf9b2185292708bcb4431d09&chksm=fc04c5e5cb734cf32b0ea6bc85f58e2b9ddfa78586dd06246089aec01335ab0e205b7d5df08d&token=1485948874&lang=zh_CN#rd)
中，我们学习了令牌桶限流算法的原理，下面我们就探讨一下，如果将`RateLimiter`扩展，让它支持集群限流，会遇到哪些问题。

&emsp;`RateLimiter`会维护两个关键的参数`nextFreeTicketMicros`和`storedPermits`，它们分别是下一次填充时间和当前存储的令牌数。当`RateLimiter`的`acquire`函数被调用时，也就是有线程希望获取令牌时，`RateLimiter`会对比当前时间和`nextFreeTicketMicros`，根据二者差距，刷新`storedPermits`，然后再判断更新后的`storedPermits`是否足够，足够则直接返回，否则需要等待直到令牌足够(Guava RateLimiter的实现比较特殊，并不是当前获取令牌的线程等待，而是下一个获取令牌的线程等待)。

&emsp;由于要支持集群限流，所以`nextFreeTicketMicros`和`storedPermits`这两个参数不能只存在JVM的内存中，必须有一个集中式存储的地方。而且，由于算法要先获取两个参数的值，计算后在更新两个数值，这里涉及到竞态限制，必须要处理并发问题。

&emsp;集群限流由于会面对相比单机更大的流量冲击，所以一般不会进行线程等待，而是直接进行丢弃，因为如果让拿不到令牌的线程进行睡眠，会导致大量的线程堆积，线程持有的资源也不会释放，反而容易拖垮服务器。

### Redis和Lua
![](/images/19_1221/6_image1.webp)

&emsp;分布式限流本质上是一个集群并发问题，Redis单进程单线程的特性，天然可以解决分布式集群的并发问题。所以很多分布式限流都基于Redis，比如说Spring Cloud的网关组件Gateway。

&emsp;Redis执行Lua脚本会以原子性方式进行，单线程的方式执行脚本，在执行脚本时不会再执行其他脚本或命令。并且，Redis只要开始执行Lua脚本，就会一直执行完该脚本再进行其他操作，所以**Lua脚本中不能进行耗时操作**。使用Lua脚本，还可以减少与Redis的交互，减少网络请求的次数。

&emsp;Redis中使用Lua脚本的场景有很多，比如说分布式锁，限流，秒杀等，总结起来，下面两种情况下可以使用Lua脚本:

- 使用 Lua 脚本实现原子性操作的CAS，避免不同客户端先读Redis数据，经过计算后再写数据造成的并发问题。
- 前后多次请求的结果有依赖时，使用 Lua 脚本将多个请求整合为一个请求。

&emsp;但是使用Lua脚本也有一些注意事项：
- 要保证安全性，在 Lua 脚本中不要定义自己的全局变量，以免污染 Redis内嵌的Lua环境。因为Lua脚本中你会使用一些预制的全局变量，比如说`redis.call()`
- 要注意 Lua 脚本的时间复杂度，Redis 的单线程同样会阻塞在 Lua 脚本的执行中。
- 使用 Lua 脚本实现原子操作时，要注意如果 Lua 脚本报错，之前的命令无法回滚，这和Redis所谓的事务机制是相同的。
- 一次发出多个 Redis 请求，但请求前后无依赖时，使用 pipeline，比 Lua 脚本方便。
- Redis要求单个Lua脚本操作的key必须在同一个Redis节点上。解决方案可以看下文对Gateway原理的解析。

### 性能测试

&emsp;Redis虽然以单进程单线程模型进行操作，但是它的性能却十分优秀。总结来说，主要是因为:
- 绝大部分请求是纯粹的内存操作
- 采用单线程,避免了不必要的上下文切换和竞争条件
- 内部实现采用非阻塞IO和epoll，基于epoll自己实现的简单的事件框架。epoll中的读、写、关闭、连接都转化成了事件，然后利用epoll的多路复用特性，绝不在io上浪费一点时间。

&emsp;所以，在集群限流时使用Redis和Lua的组合并不会引入过多的性能损耗。我们下面就简单的测试一下，顺便熟悉一下涉及的Redis命令。

```
# test.lua脚本的内容
local test = redis.call("get", "test")
local time = redis.call("get", "time")
redis.call("setex", "test", 10, "xx")
redis.call("setex", "time", 10, "xx")
return {test, time}

# 将脚本导入redis，之后调用不需再传递脚本内容
redis-cli -a 082203 script load "$(cat test.lua)"
"b978c97518ae7c1e30f246d920f8e3c321c76907"
# 使用redis-benchmark和evalsha来执行lua脚本
redis-benchmark -a 082203 -n 1000000 evalsha b978c97518ae7c1e30f246d920f8e3c321c76907 0 
======
1000000 requests completed in 20.00 seconds
50 parallel clients
3 bytes payload
keep alive: 1

93.54% <= 1 milliseconds
99.90% <= 2 milliseconds
99.97% <= 3 milliseconds
99.98% <= 4 milliseconds
99.99% <= 5 milliseconds
100.00% <= 6 milliseconds
100.00% <= 7 milliseconds
100.00% <= 7 milliseconds
49997.50 requests per second
```
&emsp;通过上述简单的测试，我们可以发现本机情况下，使用Redis执行Lua脚本的性能极其优秀，一百万次执行，99.99%在5毫秒以下。

&emsp;本来想找一下官方的性能数据，但是针对Redis + Lua的性能数据较少，只找到了几篇个人博客，感兴趣的同学可以去探索。[这篇文章](https://www.fuwuqizhijia.com/redis/201704/60935.html)有Lua和zadd的性能比较(具体数据请看原文，链接缺失的话，请看文末)。


> 以上lua脚本的性能大概是zadd的70%-80%，但是在可接受的范围内，在生产环境可以使用。负载大概是zadd的1.5-2倍，网络流量相差不大，IO是zadd的3倍，可能是开启了AOF，执行了三次操作。

### Spring Cloud Gateway的限流实现
![](/images/19_1221/6_image2.webp)

&emsp;`Gateway`是微服务架构`Spring Cloud`的网关组件，它基于Redis和Lua实现了令牌桶算法的限流功能，下面我们就来看一下它的原理和细节吧。

&emsp;`Gateway`基于Filter模式，提供了限流过滤器`RequestRateLimiterGatewayFilterFactory`。只需在其配置文件中进行配置，就可以使用。具体的配置感兴趣的同学自行学习，我们直接来看它的实现。

&emsp;`RequestRateLimiterGatewayFilterFactory`依赖`RedisRateLimiter`的`isAllowed`函数来判断一个请求是否要被限流抛弃。

```
public Mono<Response> isAllowed(String routeId, String id) {
        //routeId是ip地址，id是使用KeyResolver获取的限流维度id，比如说基于uri,IP或者用户等等。
	Config routeConfig = loadConfiguration(routeId);
	// 每秒能够通过的请求数
	int replenishRate = routeConfig.getReplenishRate();
	// 最大流量
	int burstCapacity = routeConfig.getBurstCapacity();
	try {
	    // 组装Lua脚本的KEY
		List<String> keys = getKeys(id);
		// 组装Lua脚本需要的参数，1是指一次获取一个令牌
		List<String> scriptArgs = Arrays.asList(replenishRate + "",
				burstCapacity + "", Instant.now().getEpochSecond() + "", "1");
		// 调用Redis，tokens_left = redis.eval(SCRIPT, keys, args)
		Flux<List<Long>> flux = this.redisTemplate.execute(this.script, keys,
				scriptArgs);
	..... // 省略			
}
static List<String> getKeys(String id) {
	String prefix = "request_rate_limiter.{" + id;
	String tokenKey = prefix + "}.tokens";
	String timestampKey = prefix + "}.timestamp";
	return Arrays.asList(tokenKey, timestampKey);
}				
```
&emsp;需要注意的是`getKeys`函数的prefix包含了"{id}"，这是为了解决Redis集群键值映射问题。Redis的KeySlot算法中，如果key包含{}，就会使用第一个{}内部的字符串作为hash key，这样就可以保证拥有同样{}内部字符串的key就会拥有相同slot。Redis要求单个Lua脚本操作的key必须在同一个节点上，但是Cluster会将数据自动分布到不同的节点，使用这种方法就解决了上述的问题。

&emsp;然后我们来看一下Lua脚本的实现，该脚本就在Gateway项目的resource文件夹下。它就是如同`Guava`的`RateLimiter`一样，实现了令牌桶算法，只不过不在需要进行线程休眠，而是直接返回是否能够获取。

```
local tokens_key = KEYS[1]   -- request_rate_limiter.${id}.tokens 令牌桶剩余令牌数的KEY值
local timestamp_key = KEYS[2] -- 令牌桶最后填充令牌时间的KEY值

local rate = tonumber(ARGV[1])  -- replenishRate 令令牌桶填充平均速率
local capacity = tonumber(ARGV[2]) -- burstCapacity 令牌桶上限
local now = tonumber(ARGV[3]) -- 得到从 1970-01-01 00:00:00 开始的秒数
local requested = tonumber(ARGV[4]) -- 消耗令牌数量，默认 1 

local fill_time = capacity/rate   -- 计算令牌桶填充满令牌需要多久时间
local ttl = math.floor(fill_time*2)  -- *2 保证时间充足


local last_tokens = tonumber(redis.call("get", tokens_key)) 
-- 获得令牌桶剩余令牌数
if last_tokens == nil then  -- 第一次时，没有数值，所以桶时满的
  last_tokens = capacity
end

local last_refreshed = tonumber(redis.call("get", timestamp_key)) 
-- 令牌桶最后填充令牌时间
if last_refreshed == nil then
  last_refreshed = 0
end

local delta = math.max(0, now-last_refreshed)  
-- 获取距离上一次刷新的时间间隔
local filled_tokens = math.min(capacity, last_tokens+(delta*rate)) 
-- 填充令牌，计算新的令牌桶剩余令牌数 填充不超过令牌桶令牌上限。

local allowed = filled_tokens >= requested      
local new_tokens = filled_tokens
local allowed_num = 0
if allowed then
-- 若成功，令牌桶剩余令牌数(new_tokens) 减消耗令牌数( requested )，并设置获取成功( allowed_num = 1 ) 。
  new_tokens = filled_tokens - requested
  allowed_num = 1
end       

-- 设置令牌桶剩余令牌数( new_tokens ) ，令牌桶最后填充令牌时间(now) ttl是超时时间？
redis.call("setex", tokens_key, ttl, new_tokens)
redis.call("setex", timestamp_key, ttl, now)

-- 返回数组结果
return { allowed_num, new_tokens }
```

### 后记
&emsp;Redis的主从异步复制机制可能丢失数据，出现限流流量计算不准确的情况，当然限流毕竟不同于分布式锁这种场景，对于结果的精确性要求不是很高，即使多流入一些流量，也不会影响太大。
&emsp;正如Martin在他质疑Redis分布式锁RedLock文章中说的，Redis的数据丢弃了也无所谓时再使用Redis存储数据。
> I think it’s a good fit in situations where you want to share some transient, approximate, fast-changing data between servers, and where it’s not a big deal if you occasionally lose that data for whatever reason

&emsp;接下来我们回来学习阿里开源的分布式限流组件`sentinel`，希望大家持续关注。

![](/images/logo.png)


### 参考

- https://www.cnblogs.com/itrena/p/5926878.html
- 压测的文章：https://www.fuwuqizhijia.com/redis/201704/60935.html
- https://blog.csdn.net/forezp/article/details/85081162
- https://blog.csdn.net/xixingzhe2/article/details/86167859
- Matin RedLock http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

