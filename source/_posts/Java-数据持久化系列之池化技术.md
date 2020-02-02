---
title: Java 数据持久化系列之池化技术
tags: persistent
category: java persistent
abbrlink: d5009955
date: 2020-02-02 21:48:17
---

在上一篇文章[《Java 数据持久化系列之JDBC》](http://remcarpediem.net/article/8c16d4e4/)中，我们了解到使用 JDBC 创建 Connection 可以执行对应的SQL，但是创建 Connection 会消耗很多资源，所以 Java 持久化框架中往往不直接使用 JDBC，而是在其上建立数据库连接池层。

今天我们就先来了解一下池化技术的必要性、原理；然后使用 Apache-common-Pool2实现一个简单的数据库连接池；接着通过实验，对比简单连接池、HikariCP、Druid 等数据库连接池的性能数据，分析实现高性能数据库连接池的关键；最后分析 Pool2 的具体源代码实现。

![](https://upload-images.jianshu.io/upload_images/623378-28c417a8451c7c31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 对象不是你想要，想要就能要
你我单身狗们经常调侃可以随便 New 出一个对象，用完就丢。但是有些对象创建的代价比较大，比如线程、tcp连接、数据库连接等对象。对于这些创建耗时较长，或者资源占用较大(占据操作系统资源，比如说线程，网络连接等)的对象，往往会引入池化来管理，减少频繁创建对象的次数，避免创建对象时的耗时，提高性能。

我们就以数据库连接 Connection 对象为例，详细说明一下创建该对象花费的时间和资源。下面是MySQL Driver 创建 Connection 对象的方法，在调用 connect 方法创建 Connection 时，会与 MySQL 进行网络通讯，建立 TCP 连接，这是极其消耗时间的。

``` java
connection = driver.connect(URL, props);
```

### 使用 Apache-Common-Pool2实现简易数据库连接池

下面，我们以 Apache-Common-Pool2为例来看一下池化技术相关的抽象结构。

首先了解一下Pool2中三元一体的 ObjectPool，PooledObject 和 PooledObjectFactory，对他们的解释如下：
- ObjectPool 就是对象池，提供了 `borrowObject` 和 `returnObject` 等一系列函数。
- PooledObject 是池化对象的封装类，负责记录额外信息，比如说对象状态，对象创建时间，对象空闲时间，对象上次使用时间等。
- PooledObjectFactory 是负责管理池化对象生命周期的工厂类，提供 `makeObject`，`destroyObject`，`activateObject` 和 `validateObject` 等一系列函数。

上述三者都有其基础的实现类，分别是 GenericObjectPool，DefaultPooledObject 和 BasePooledObjectFactory。上一节实验中的 SimpleDatasource 就是使用上述类实现的。

首先，你要实现一个继承 BasePooledObjectFactory 的工厂类，提供管理池化对象生命周期的具体方法：
- makeObject：创建池化对象实例，并且使用 PooledObject 将其封装。
- validateObject：验证对象实例是否安全或者可用，比如说 Connection 是否还保存连接状态。
- activateObject：将池返回的对象实例进行重新初始化，比如说设置 Connection是否默认AutoCommit等。
- passivateObject：将返回给池的对象实例进行反初始化，比如说 Connection 中未提交的事务进行 Rollback等。
- destroyObject：销毁不再被池需要的对象实例，比如说 Connection不再被需要时，调用其 close 方法。

具体的实现源码如下所示，每个方法都有详细的注释。

```
public class SimpleJdbcConnectionFactory extends BasePooledObjectFactory<Connection> {
    ....
    @Override
    public Connection create() throws Exception {
        // 用于创建池化对象
        Properties props = new Properties();
        props.put("user", USER_NAME);
        props.put("password", PASSWORD);
        Connection connection = driver.connect(URL, props);
        return connection;
    }

    @Override
    public PooledObject<Connection> wrap(Connection connection) {
        // 将池化对象进行封装，返回DefaultPooledObject，这里也可以返回自己实现的PooledObject
        return new DefaultPooledObject<>(connection);
    }

    @Override
    public PooledObject<Connection> makeObject() throws Exception {
        return super.makeObject();
    }

    @Override
    public void destroyObject(PooledObject<Connection> p) throws Exception {
        p.getObject().close();
    }

    @Override
    public boolean validateObject(PooledObject<Connection> p) {
        // 使用 SELECT 1 或者其他sql语句验证Connection是否可用，ConnUtils代码详见Github中的项目
        try {
            ConnUtils.validateConnection(p.getObject(), this.validationQuery);
        } catch (Exception e) {
            return false;
        }
        return true;
    }


    @Override
    public void activateObject(PooledObject<Connection> p) throws Exception {
        Connection conn = p.getObject();
        // 对象被借出，需要进行初始化，将其 autoCommit进行设置
        if (conn.getAutoCommit() != defaultAutoCommit) {
            conn.setAutoCommit(defaultAutoCommit);
        }
    }

    @Override
    public void passivateObject(PooledObject<Connection> p) throws Exception {
        // 对象被归还，进行回收或者处理，比如将未提交的事务进行回滚
        Connection conn = p.getObject();
        if(!conn.getAutoCommit() && !conn.isReadOnly()) {
            conn.rollback();
        }
        conn.clearWarnings();
        if(!conn.getAutoCommit()) {
            conn.setAutoCommit(true);
        }

    }
}

```


接着，你就可以使用 BasePool 来从池中获取对象，使用后归还给池。

``` java
Connection connection = pool.borrowObject(); // 从池中获取连接对象实例
Statement statement = connection.createStatement();
statement.executeQuery("SELECT * FROM activity");
statement.close();
pool.returnObject(connection); // 使用后归还连接对象实例到池中
```
如上，我们就使用 Apache Common Pool2 实现了一个简易的数据库连接池。下面，我们先来使用 benchmark 验证一下这个简易数据库连接池的性能，再分析 Pool2 的具体源码实现，

### 性能试验

至此，我们分析完了 Pool2的相关原理和实现，下面就修改 Hikari-benchmark 对我们编写的建议数据库连接池进行性能测试。修改后的 benchmark 的地址为 https://github.com/ztelur/HikariCP-benchmark。

![](https://upload-images.jianshu.io/upload_images/623378-457c7848c1560aa1.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到 hikari 和 Druid 两个数据库连接池的性能是最优的，而我们的简易数据库连接池性能排在末尾。在后续系列文章中会对比我们的简易数据库分析 Hikari 和 Druid 高性能的原因。下面我们先来看一下简易数据库连接池的具体实现。

### Apache Common Pool2 源码分析

我们来简要分析 Pool2 的源码( 2.8.0版本 )实现，了解池化技术的基本原理，为后续了解和分析 HikariCP 和 Druid 打下基础，三者在设计思路具有互通之处。

通过前边的实例，我们知道通过 `borrowObject` 和 `returnObject` 从对象池中接取或者归还对象，进行这些操作时，封装实例 PooledObject 的状态也会发生变化，下面就沿着 PooledObject 状态机的状态变化路线来讲解相关的代码实现。


![PooledObject 状态机示意图](https://upload-images.jianshu.io/upload_images/623378-3dd8857f684819cb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上图是 PooledObject 的状态机示意图，蓝色元素代表状态，红色代表 ObjectPool的相关方法。PooledObject 的状态有 IDLE、ALLOCATED、RETURNING、ABANDONED、INVALID、EVICTION 和 EVICTION_RETURN_TO_HEAD(所有状态定义在 PooledObjectState 类中，有些状态暂时未被使用，这里不进行说明)。

主要涉及三部分的状态变化，分别是 1、2、3的借出归还状态变化，4，5的标记抛弃删除状态变化以及6,7,8的检测驱除状态变化。后续会分小节详细介绍这三部分的状态变化。

在这些状态变化过程中，不仅涉及 ObjectPool 的方法，也会调用 PooledObjectFactory 的方法进行相关操作。

![PooledOjbectFactory方法示意图](https://upload-images.jianshu.io/upload_images/623378-dab3557ec3f9fee4.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图表明了在 PooledObject 状态变化过程中涉及的 PooledObjectFactory 的方法。按照前文对 PooledObjectFactory 方法的描述，可以很容易的对应起来。比如说，在编号 1 的对象被借出过程中，先调用 invalidateObject 判断对象可用性，然后调用 activeObject 将对象默认配置初始化。

#### 借出归还状态变化
我们从 GenericObjectPool 的 borrowObject 方法开始了解。该方法可以传入最大等待时间为参数，如果不传则使用配置的默认最大等待时间，borrowObject 的源码如下所示(为了可读性，对代码进行删减)。

```
public T borrowObject(final long borrowMaxWaitMillis) throws Exception {
    // 1 根据 abandonedConfig 和其他检测判断是否要调用 removeAbandoned 方法进行标记删除操作
    ....
    PooledObject<T> p = null;
    // 当暂时无法获取对象时是否阻塞
    final boolean blockWhenExhausted = getBlockWhenExhausted();
    while (p == null) {
        create = false;
        // 2 先从 idleObjects 队列中获取, pollFisrt 是非阻塞的
        p = idleObjects.pollFirst();
        // 3 没有则调用 create 方法创建一个新的对象
        if (p == null) {
            p = create();
        }
        // 4 blockWhenExhausted 为true，则根据 borrowMaxWaitMillis 进行阻塞操作
        if (blockWhenExhausted) {
            if (p == null) {
                if (borrowMaxWaitMillis < 0) {
                    p = idleObjects.takeFirst(); // 阻塞到获取对象为止
                } else {
                    p = idleObjects.pollFirst(borrowMaxWaitMillis,
                            TimeUnit.MILLISECONDS); // 阻塞到最大等待时间或者获取到对象
                }
            }
        }
        // 5 调用 allocate 进行状态变化
        if (!p.allocate()) {
            p = null;
        }
        if (p != null) {
            // 6 调用 activateObject 进行对象默认初始化，如果出现问题则调用 destroy 
            factory.activateObject(p);
            // 7 如果配置了 TestOnBorrow，则调用 validateObject 进行可用性校验，如果不通过则调用 destroy
            if (getTestOnBorrow()) {
                validate = factory.validateObject(p);
            }
        }
    }
    return p.getObject();
}

```
borrowObject 方法主要做了五步操作：
- 第一步是根据配置判断是否要调用 removeAbandoned 方法进行标记删除操作，这个后续小节再细讲。
- 第二步是尝试获取或创建对象，由源码中2，3，4 步骤组成。
- 第三步是调用 allocate 进行状态变更，转换为 ALLOCATED 状态，如源码中的 5 步骤。
- 第四步是调用 factory 的 activateObject 进行对象的初始化，如果出错则调用 destroy 方法销毁对象，如源码中的 6 步骤。
- 第五步是根据 TestOnBorrow 配置调用 factory 的 validateObject 进行对象可用性分析，如果不可用，则调用 destroy 方法销毁对象，如源码中的 7 步骤。

![示意图](https://upload-images.jianshu.io/upload_images/623378-6ae16a1996e4e3cd.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


我们对第二步进行一下细致的分析。idleObjects 是存储着所有 IDLE状态 (也有可能是 EVICTION 状态) PooledObject 的 LinkedBlockingDeque 对象。第二步中先调用其 pollFirst 方法从队列头获取 PooledObject，如果未获取到则调用 create 方法创建一个新的。

create 也可能未创建成功，则当 blockWhenExhausted 为 true 时，未获取到对象需要一直阻塞，所以根据最大等待时间 borrowMaxWaitMillis 来调用 takeFirst 或者 pollFirst(time) 方法进行阻塞式获取；当 blockWhenExhausted 为 false 时，则直接抛出异常返回。
 
create 方法会判断当前状况下是否应该创建新的对象，主要是要防止创建的对象数量超过最大池对象数量。如果可以创建新对象，则调用 PooledObjectFactory 的 makeObject 方法进行新对象创建，然后根据 testOnCreate 配置来判断是否调用 validateObject 方法进行校验，源码如下所示。

```
private PooledObject<T> create() throws Exception {
    int localMaxTotal = getMaxTotal(); // 获取池对象最大数量
    final long localStartTimeMillis = System.currentTimeMillis();
    final long localMaxWaitTimeMillis = Math.max(getMaxWaitMillis(), 0); // 获取最大等待时间
    Boolean create = null;
    // 一直等待到 create 被赋值，true代表要创建新对象，false代表不能创建
    while (create == null) {
        synchronized (makeObjectCountLock) {
            final long newCreateCount = createCount.incrementAndGet();
            if (newCreateCount > localMaxTotal) {
                // pool已经满或者正在创建的足够达到最大数量的对象。
                createCount.decrementAndGet();
                if (makeObjectCount == 0) {
                    // 目前没有其他的 makeObject 方法被调用，直接返回false
                    create = Boolean.FALSE;
                } else {
                    // 目前有其他的 makeObject 方法被调用，但是可能失败，所以等待一段时间再试试
                    makeObjectCountLock.wait(localMaxWaitTimeMillis);
                }
            } else {
                // pool未满 可以创建对象。
                makeObjectCount++;
                create = Boolean.TRUE;
            }
        }

        // 执行超过 maxWaitTimeMillis 则返回false
        if (create == null &&
            (localMaxWaitTimeMillis > 0 &&
                System.currentTimeMillis() - localStartTimeMillis >= localMaxWaitTimeMillis)) {
            create = Boolean.FALSE;
        }
    }
    // 如果 create 为false，返回 NULL
    if (!create.booleanValue()) {
        return null;
    }

    final PooledObject<T> p;
    try {
        // 调用 factory 的 makeObject 进行对象创建，并且按照 testOnCreate 配置调用 validateObject 方法
        p = factory.makeObject();
        if (getTestOnCreate() && !factory.validateObject(p)) {
            // 这里代码有问题，校验不通过的对象没有进行销毁？
            createCount.decrementAndGet();
            return null;
        }
    } catch (final Throwable e) {
        createCount.decrementAndGet();
        throw e;
    } finally {
        synchronized (makeObjectCountLock) {
            // 减少 makeObjectCount
            makeObjectCount--;
            makeObjectCountLock.notifyAll();
        }
    }
    allObjects.put(new IdentityWrapper<>(p.getObject()), p);
    return p;
}
```
需要注意的是 create 方法创建的对象并没有第一时间加入到 idleObjects 队列中，该对象将会在后续使用完毕调用 returnObject 方法时才会加入到队列中。

接下来，我们看一下 returnObject 方法的实现。该方法主要做了六步操作：
- 第一步是调用 markReturningState 方法将状态变更为 RETURNING。
- 第二步是根据 testOnReturn 配置调用 PooledObjectFactory  的 validateObject 方法进行可用性校验。如果未通过校验，则调用 destroy 消耗该对象，然后调用 ensureIdle 确保池中有 IDLE 状态对象可用，如果没有会调用 create 方法创建新的对象。
- 第三步是调用 PooledObjectFactory 的 passivateObject 方法进行反初始化操作。
- 第四步是调用 deallocate 将状态变更为 IDLE。
- 第五步是检测是否超过了最大 IDLE 对象数量，如果超过了则销毁当前对象。
- 第六步是根据 LIFO (last in, first out) 配置将对象放置到队列的首部或者尾部。
```
public void returnObject(final T obj) {
    final PooledObject<T> p = allObjects.get(new IdentityWrapper<>(obj));
    // 1 将状态转换为 RETURNING
    markReturningState(p);

    final long activeTime = p.getActiveTimeMillis();
    // 2 根据配置，对实例进行可用性校验
    if (getTestOnReturn() && !factory.validateObject(p)) {
        destroy(p);
        // 因为删除了一个对象，需要确保池内还有对象，如果没有改方法会创建新对象
        ensureIdle(1, false); 
        updateStatsReturn(activeTime);
        return;
    }
    // 3 调用 passivateObject 将对象反初始化。
    try {
        factory.passivateObject(p);
    } catch (final Exception e1) {
         .... // 和上边 validateObject 校验失败相同操作。
    }
    // 4 将状态变更为 IDLE
    if (!p.deallocate()) {
        throw new IllegalStateException(
                "Object has already been returned to this pool or is invalid");
    }

    final int maxIdleSave = getMaxIdle();
    // 5 如果超过最大 IDLE 数量，则进行销毁
    if (isClosed() || maxIdleSave > -1 && maxIdleSave <= idleObjects.size()) {
        .... // 同上边 validateObject 校验失败相同操作。
    } else {
        // 6 根据 LIFO 配置，将归还的对象放置在队列首部或者尾部。 这边源码拼错了。
        if (getLifo()) {
            idleObjects.addFirst(p);
        } else {
            idleObjects.addLast(p);
        }
    }
    updateStatsReturn(activeTime);
}

```

下图介绍了第六步两种入队列的场景，LIFO 为 true 时防止在队列头部；LIFO 为 false 时，防止在队列尾部。要根据不同的池化对象选择不同的场景。但是放置在尾部可以避免并发热点，因为借对象和还对象都需要操作队列头，需要进行并发控制。

![LIFO示意图](https://upload-images.jianshu.io/upload_images/623378-f12e14496e70398a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 标记删除状态变化

标记删除状态变化操作主要通过 removeAbandoned 实现，它主要是检查已经借出的对象是否需要删除，防止对象被借出长时间未使用或者归还所导致的池对象被耗尽的情况。

removeAbandoned 根据 AbandonedConfig 可能会在 borrowObject 或者 检测驱除对象的 evict 方法执行时被调用。

```
public T borrowObject(final long borrowMaxWaitMillis) throws Exception {
    
    final AbandonedConfig ac = this.abandonedConfig;
    // 当配置了 removeAbandonedOnBorrow 并且 当前 idle 对象数量少于2，活跃对象数量只比最大对象数量少3.
    if (ac != null && ac.getRemoveAbandonedOnBorrow() &&
            (getNumIdle() < 2) &&
            (getNumActive() > getMaxTotal() - 3) ) {
        removeAbandoned(ac);
    }
    ....
}

public void evict() throws Exception {
    ....
    final AbandonedConfig ac = this.abandonedConfig;
        // 设置了 removeAbandonedOnMaintenance
        if (ac != null && ac.getRemoveAbandonedOnMaintenance()) {
            removeAbandoned(ac);
        }
}
```
removeAbandoned 使用典型的标记删除策略：标记阶段是先对所有的对象进行遍历，如果该对象是 ALLOCATED 并且上次使用时间已经超过超时时间，则将其状态变更为 ABANDONED 状态，并加入到删除队列中；删除阶段则遍历删除队列，依次调用 invalidateObject 方法删除并销毁对象。

```
private void removeAbandoned(final AbandonedConfig ac) {
    // 收集需要 abandoned 的对象
    final long now = System.currentTimeMillis();
    // 1 根据配置的时间计算超时时间
    final long timeout =
            now - (ac.getRemoveAbandonedTimeout() * 1000L);
    final ArrayList<PooledObject<T>> remove = new ArrayList<>();
    final Iterator<PooledObject<T>> it = allObjects.values().iterator();
    while (it.hasNext()) {
        final PooledObject<T> pooledObject = it.next();
        // 2 遍历所有的对象，如果它是已经分配状态，并且该对象的最近一次使用时间小于超时时间
        synchronized (pooledObject) {
            if (pooledObject.getState() == PooledObjectState.ALLOCATED &&
                    pooledObject.getLastUsedTime() <= timeout) {
                // 3 将对象状态更改为 ABANDONED,并加入到删除队列
                pooledObject.markAbandoned();
                remove.add(pooledObject);
            }
        }
    }

    // 4 遍历删除队列
    final Iterator<PooledObject<T>> itr = remove.iterator();
    while (itr.hasNext()) {
        final PooledObject<T> pooledObject = itr.next();
        // 5 调用 invalidateObject 方法删除对象
        invalidateObject(pooledObject.getObject());
    }
}
```
invalidateObject 方法直接调用了 destroy 方法，destroy 方法在上边的源码分析中也反复出现，它主要进行了四步操作：
- 1 将对象状态变更为 INVALID。
- 2 将对象从队列和集合中删除。
- 3 调用 PooledObjectFactory 的 destroyObject 方法销毁对象。
- 4 更新统计数据

```
private void destroy(final PooledObject<T> toDestroy) throws Exception {
    // 1 将状态变更为 INVALID
    toDestroy.invalidate();
    // 2 从队列和池中删除
    idleObjects.remove(toDestroy);
    allObjects.remove(new IdentityWrapper<>(toDestroy.getObject()));
    // 3 调用 destroyObject 回收对象
    try {
        factory.destroyObject(toDestroy);
    } finally {
        // 4 更新统计数据
        destroyedCount.incrementAndGet();
        createCount.decrementAndGet();
    }
}
```

#### 检测驱除状态变化

检测驱除状态变化主要由 evict 方法操作，在后台线程中独立完成，主要检测池中的 IDLE 状态的空闲对象是否需要驱除，超时时间通过 EvictionConfig 进行配置。

驱逐者 Evictor,在 BaseGenericObjectPool 中定义，本质是由 java.util.TimerTask 定义的定时任务。

```
final void startEvictor(final long delay) {
    synchronized (evictionLock) {
        if (delay > 0) {
            // 定时执行 evictor 线程
            evictor = new Evictor();
            EvictionTimer.schedule(evictor, delay, delay);
        }
    }
}
```

在 Evictor 线程中会调用 evict 方法，该方法主要是遍历所有的 IDLE 对象，然后对每个对象执行检测驱除操作，具体源码如下所示：
- 调用 startEvictionTest 方法将状态更改为 EVICTED。
- 根据驱除策略和对象超时时间判断是否要驱除。
- 如果需要被驱除则调用 destroy 方法销毁对象。
- 如果设置了 testWhileIdle 则调用 PooledObject 的 validateObject 进行可用性校验。
- 调用 endEvictionTest 将状态更改为 IDLE。

```
public void evict() throws Exception {
    if (idleObjects.size() > 0) {
        ....
        final EvictionPolicy<T> evictionPolicy = getEvictionPolicy();
        synchronized (evictionLock) {
            for (int i = 0, m = getNumTests(); i < m; i++) {
                // 1 遍历所有 idle 的对象
                try {
                    underTest = evictionIterator.next();
                } catch (final NoSuchElementException nsee) {
                }
                // 2 调用 startEvictionTest 将状态变更为 EVICTED
                if (!underTest.startEvictionTest()) {
                    continue;
                }
                // 3 根据驱除策略判断是否要驱除
                boolean evict = evictionPolicy.evict(evictionConfig, underTest,
                        idleObjects.size());

                if (evict) {
                    // 4 进行驱除
                    destroy(underTest);
                    destroyedByEvictorCount.incrementAndGet();
                } else {
                    // 5 如果需要检测，则进行可用性检测
                    if (testWhileIdle) {
                        factory.activateObject(underTest);
                        factory.validateObject(underTest));
                        factory.passivateObject(underTest);
                        }
                    // 5 变更状态为 IDLE
                    if (!underTest.endEvictionTest(idleObjects)) {
                    }
                }
            }
        }
    }
    .... // abandoned 相关的操作
}
```


### 后记

后续会分析 Hikari 和 Druid 数据库连接池的实现，请大家多多关注。


[个人博客，欢迎来玩](http://remcarpediem.net/article/8c16d4e4/)

![](https://upload-images.jianshu.io/upload_images/623378-289642734e6eb81b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 参考

- [https://zhuanlan.zhihu.com/p/32204303](https://zhuanlan.zhihu.com/p/32204303)
- [https://juejin.im/post/5af026a06fb9a07ac47ff282](https://juejin.im/post/5af026a06fb9a07ac47ff282)
- 高性能连接池的技术细节 https://yq.aliyun.com/articles/59497
- apache common的通用池 http://www.victorchu.info/2019/01/05/%E4%BB%8Eapache-common-pool%E7%9C%8B%E5%A6%82%E4%BD%95%E5%86%99%E4%B8%80%E4%B8%AA%E9%80%9A%E7%94%A8%E6%B1%A0/
- 如何设计一个连接池：commons-pool2源码分 https://throwsnew.com/2017/06/12/commons-pool/
- https://zhuanlan.zhihu.com/p/32204303
- https://yq.aliyun.com/articles/59497](https://yq.aliyun.com/articles/59497