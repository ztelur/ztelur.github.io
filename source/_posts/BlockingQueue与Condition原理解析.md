---
title: BlockingQueue与Condition原理解析
tags:
  - JUC
categories:
  - 并发
abbrlink: 815f6f66
date: 2017-05-03 10:07:45
---
 我在前段时间写了一篇关于AQS的[文章](http://remcarpediem.com/2017/04/06/AbstractQueuedSynchronizer%E8%B6%85%E8%AF%A6%E7%BB%86%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)，在文章里边我说几乎所有在`JUC`包中的所有多线程相关的类都和`AQS`相关，今天我就在这里总结一下另一个依赖于`AQS`来实现的同步工具类：`BlockingQueue`。我们主要以`ArrayBlockingQueue`为主来分析相关的源码。

### 阻塞队列
 相信大多数同学都是在学习线程池相关知识时了解到阻塞队列的概念的。知道各种类型的阻塞队列对线程池初始化时的影响。在java doc中这样定义阻塞队列。当从阻塞队列获取元素但是队列为空时，当前线程会阻塞直到另一个线程向阻塞队列中添加一个元素；类似的，当向一个阻塞队列加入元素时，如果队列已经满了，当前线程也会阻塞知道另外一个线程从队列中读取一个元素。阻塞队列一般都是FIFO,用来实现生产者和消费者模式。阻塞队列的方法通过四种不同的方式来处理操作无法被立即完成的情况，这四种情况分别为抛出异常，返回特殊值(null或在是false),阻塞当前线程直到执行结束，最后一种是只阻塞固定时间，然后还未执行成功就放弃操作。这些方法都总结在下边这种表中了。

![BlockingQueue](http://7xrxif.com1.z0.glb.clouddn.com/2017416-condition-BlockingQueue.png)

 我们就只分析`put`和`take`方法。
### put和take函数
 我们都知道，使用同步队列可以很轻松的实现生产者-消费者模式，其实，同步队列就是按照生产者-消费者的模式来实现的，我们可以将`put`函数看作生产者的操作，`take`是消费者的操作。
 `put`函数会在队列末尾添加元素，如果队列已经满了，无法添加元素的话，就一直阻塞等待到可以加入为止。函数的源码如下所示。
``` java
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly(); //先获得锁
        try {
            while (count == items.length) 
            //如果队列满了，就NotFull这个condition对象上进行等待
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
    private void enqueue(E x) {
        final Object[] items = this.items;
        items[putIndex] = x;
        //这里可以注意的是ArrayBlockingList实际上使用Array实现了一个环形数组，
       //当putIndex达到最大时，就返回到起点，继续插入,
       //当然，如果此时0位置的元素还没有被取走，
       //下次put时，就会因为cout == item.length未被阻塞。
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        //因为插入了元素，通知等待notEmpty事件的线程。
        notEmpty.signal();
    }
```
 我们会发现put函数也是使用了wait/notify的机制。与一般生产者-消费者的实现方式不同，同步队列使用`ReentrantLock`和`Condition`相结合的先获得锁，再等待的机制；而不是`synchronized`和`Object.wait`的机制。这里的区别我们下一节再详细讲解。
 看完了生产者相关的`put`函数，我们再来看一下消费者调用的`take`函数。`take`函数在队列为空时会被阻塞，一直到阻塞队列加入了新的元素。
``` java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
            //如果队列为空，那么在notEmpty对象上等待，
            //当put函数调用时，会调用notEmpty的notify进行通知。
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
    private E dequeue() {
        E x = (E) items[takeIndex];
        items[takeIndex] = null; //取出takeIndex位置的元素
        if (++takeIndex == items.length)
            //如果到了尾部，将指针重新调整到头部
            takeIndex = 0;
        count--;
        ....
        //通知notFull对象上等待的线程
        notFull.signal();
        return x;
    }
```

### Condition.await和Object.wait
 我们发现`ArrayBlockingList`并没有使用`Object.wait`，而是使用的`Condition.await`，这是为什么呢？其中又有哪些原因呢？
 `Condition`对象可以提供和`Object`的`wait`和`notify`一样的行为，但是后者必须使用`synchronized`这个内置的monitor锁，而`Condition`使用的是`RenentranceLock`。这两种方式在阻塞等待时都会将相应的锁释放掉，但是`Condition`的等待可以中断，这是二者唯一的区别。
＆emsp;Condition的流程大致如下边两张图所示.

![await](http://7xrxif.com1.z0.glb.clouddn.com/2017416-condition-condition1.png)



![notify](http://7xrxif.com1.z0.glb.clouddn.com/2017416-condition-condition2.png)



 我们首先来看一下`await`函数的实现，详细的讲解都在代码中．

``` java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    //在condition wait队列上添加新的节点
    Node node = addConditionWaiter();
    //释放当前持有的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    //由于node在之前是添加到condition wait queue上的，现在判断这个node
    //是否被添加到Sync的获得锁的等待队列上。
    //node在condition queue上说明还在等待事件的notify,
    //notify函数会将condition queue 上的node转化到Sync的队列上。
    while (!isOnSyncQueue(node)) {
        //node还没有被添加到Sync Queue上，说明还在等待事件通知
        //所以调用park函数来停止线程执行
        LockSupport.park(this);
        //判断是否被中断,线程从park函数返回有两种情况，一种是
        //其他线程调用了unpark,另外一种是线程被中断
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //代码执行到这里，已经有其他线程调用notify函数，或则被中断，该线程可以继续执行，但是必须先
    //再次获得调用await函数时的锁．acquireQueued函数在AQS文章中做了介绍．
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
  　．．．．
}

final int fullyRelease(Node node) {
    //AQS的方法，当前已经在锁中了，所以直接操作
    boolean failed = true;
    try {
        int savedState = getState();
        //获取state当前的值，然后保存，以待以后恢复
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}

/**
 * Checks for interrupt, returning THROW_IE if interrupted
 * before signalled, REINTERRUPT if after signalled, or
 * 0 if not interrupted.
 */
private int checkInterruptWhileWaiting(Node node) {
    //中断可能发生在两个阶段中，一是在等待singla,另外一个是在获得signal之后
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}

final boolean transferAfterCancelledWait(Node node) {
    //这里要和下边的transferForSignal对应着看，这是线程中断进入的逻辑．那边是signal的逻辑
    //两边可能有并发冲突，但是成功的一方必须调用enq来进入acquire lock queue中．
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    //如果失败了，说明transferForSignal那边成功了，等待node 进入acquire lock queue
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}

```

 `signal`函数将等待事件最长时间的线程节点从等待condition的队列移动到获得lock的等待队列中．
``` java
public final void signal() {
    //
    if (!isHeldExclusively())
    //如果当前线程没有获得锁，抛出异常
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        //将Condition wait queue中的第一个node转移到acquire lock queue中．
        doSignal(first);
}

private void doSignal(Node first) {
    do {
　　 //由于生产者的signal在有消费者等待的情况下，必须要通知
        //一个消费者，所以这里有一个循环，直到队列为空
        //把first 这个node从condition queue中删除掉
        //condition queue的头指针指向node的后继节点，如果node后续节点为null,那么也将尾指针也置为null
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
    //transferForSignal将node转而添加到Sync的acquire lock 队列
}

final boolean transferForSignal(Node node) {
    //如果设置失败，说明该node已经被取消了,所以返回false,让doSignal继续向下通知其他未被取消的node
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    //将node添加到acquire lock　queue中．
    Node p = enq(node);
    int ws = p.waitStatus;
    //需要注意的是这里的node进行了转化
    //ws>0代表canceled的含义所以直接unpark线程
    //如果compareAndSetWaitStatus失败，所以直接unpark,让线程继续执行await中的
    //进行isOnSyncQueue判断的while循环,然后进入acquireQueue函数．
    //这里失败的原因可能是Lock其他线程释放掉了锁，同步设置p的waitStatus
    //如果compareAndSetWaitStatus成功了呢？那么该node就一直在acquire lock queue中
    //等待锁被释放掉再次抢夺锁，然后再unparｋ
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}

```

### 后记
&emsp;后边一篇文章主要讲解如何自己使用`AQS`来创建符合自己业务需求的锁，请大家继续关注我的文章啦．一起进步偶．
