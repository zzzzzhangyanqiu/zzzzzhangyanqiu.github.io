---
title: AbstractQueuedSynchronizer详细分析
tags:
  - Java
date: 2021-01-16
categories:
 - Java
---

### 前言

`AbstractQueuedSynchronizer`，简称`AQS`。它提供了一个框架，用于实现依赖于先进先出（`FIFO`）等待队列的阻塞锁和相关同步器（信号灯，事件等）。也可以说`AQS`就是`Java`并发的基础，有许多工具类（`Semaphore、ReentrantLock`等）都是使用`AQS`来实现的，下面我们一起来揭开它神秘的面纱，需要注意的是，`AQS`支持独占锁和共享锁两种模式，但是两个思路其实都差不多，所以这里我们只分析独占锁，感兴趣的同学可以自己尝试分析一下共享锁哦。

### 正文

#### 数据结构

在阅读源码之前，我们可以先想一下，一个锁框架一般需要提供哪些功能，下面是我的想法

1. 能够让某个线程加锁和释放锁（废话）
2. 如果锁已经被线程`A`占用，线程`B`申请锁时，可以进行阻塞等待。`A`释放锁时会唤醒之前阻塞等待的线程
3. 如果锁已经被线程`A`占用，线程`B`可以进行尝试获取锁，如果获取不到直接返回`false`，不要阻塞线程

那么`AQS`是如何实现的上述三个功能呢，带着这个问题，我们来详细分析一下它的源码

首先，比较重要的属性有三个，如下

```java
    /**
     * 当前等待队列的头结点
     */
	private transient volatile Node head;

    /**
     * 当前等待队列的尾结点
     */
    private transient volatile Node tail;

    /**
     * 标记当前锁状态，是否被占用
     */
    private volatile int state;
```

这里的**等待队列**代表的是：如果锁已经被线程`A`占用了，这时线程`B`来申请锁，会将`B`阻塞并加入到集合中等待，这里的集合就叫做**等待队列**。`state`的含义是标记当前锁状态是否被占用，可能有的同学会对这里有一些疑惑，那我直接用`boolean`类型不就行了嘛，还节省空间，之前我们提到过一次，`AQS`支持独占锁和共享锁两种模式，并且是**可重入锁**（可重入的意思是：一个线程可以对同一个锁资源进行多次加锁操作。如果不可重入，在申请锁资源的时候，就认为锁已经被占用了（但是占用锁资源的线程是自己），那么就会阻塞当前线程，然后就会陷入自己等待自己的状态，造成死锁）。这里的`state`表示的就是当前锁被重入的次数和占有当前锁资源的线程数，为了保证线程可见性，三个变量均使用`volatile`修饰。

我们可以注意到，`head`和`tail`的字段类型是`Node`，`Node`是`AQS`的一个内部类，其主要数据结构如下

```java
static final class Node {
        /** 标记结点正在共享模式下，不需要过多关注 */
        static final Node SHARED = new Node();
        /** 标记结点正在独占模式下 */
        static final Node EXCLUSIVE = null;

        /**
         * 标记当前结点的状态，取以下四个值
         */
        volatile int waitStatus;
    
    
        /** 由于超时或者中断，当前结点已经被取消 */
        static final int CANCELLED =  1;
        /** 后续结点正处于等待状态，如果当前结点上的线程被释放或者取消，需要通知后续结点 */
        static final int SIGNAL    = -1;
        /** 当前结点正在等待condition */
        static final int CONDITION = -2;
        /** 表示后续结点会传播唤醒的操作，共享模式下起作用 */
        static final int PROPAGATE = -3;

        /**
         * 当前结点的前一个结点
         */
        volatile Node prev;

        /**
         * 当前结点的下一个结点
         */
        volatile Node next;

        /**
         * 当前结点上关联的线程
         * 如果当前结点已经占有了锁资源，那么此变量会重置为null
         */
        volatile Thread thread;

        /**
         * 下一个等待条件的结点，与condition有关，这里不用过多关注
         */
        Node nextWaiter;
```

举个例子，当前对象内已经有一个线程持有锁了，其它三个线程在等待获取锁，并且其中一个线程已经中断了（可能是因为超时或者调用了`interrupt()`方法），那么当前状态如下图

![image-20201214113403978](/images/oss/PicGo/image-20201214113403978.png)

看到上面的图，我们就可以先推测一下加锁或者解锁的大概流程。下面是我的猜想

1. 加锁：将当前线程封装成为一个`Node`结点，插入到等待队列中，然后更新`tail`变量
2. 解锁：当第一个结点释放锁之后，通过`next`获取下面一个结点，更新`head`变量并对结点上的线程做唤醒操作

接下来带着我们的猜想，来分析加锁和释放锁相关的代码

#### 加锁

`AQS`中加锁相关的方法主要有两个，如下

```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

##### tryAcquire()

首先说说`tryAcquire()`方法，该方法会进行尝试加锁，并且直接返回加锁的结果（如果加锁失败要立即返回而不是阻塞），`AQS`将此方法留给了子类实现，这么说可能有些抽象，我们来看下两个子类`Semaphore.Sync`和`ReentrantLock.Sync`中的实现

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    /**
     * AQS推荐的方式是使用内部类来继承它
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
    	
    }
    
    /**
     * 这里只拿公平锁来举例，忽略其它关键代码
     */
    static final class FairSync extends Sync {
        protected final boolean tryAcquire(int acquires) {
            //获取当前线程
            final Thread current = Thread.currentThread();
            //调用AQS的getState()方法获取state状态值
            int c = getState();
            //如果为0则代码锁资源未被占用
            if (c == 0) {
                //hasQueuedPredecessors()：查询是否有其它线程已经在等待加锁了
                //compareAndSetState：调用AQS的方法，通过cas设置state
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    //设置当前拥有独占访问权的线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //如果不为0，判断是否为当前线程占用（这里是重入锁的场景）
            else if (current == getExclusiveOwnerThread()) {
                //修改state
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
}
```

这里也可以看出`ReentrantLock`是支持重入锁的，下面再来看看`Semaphore`

```java
public class Semaphore implements java.io.Serializable {
    private final Sync sync;


    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        //如果申请资源后，资源量小于0，则加锁失败
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }
    
    abstract static class Sync extends AbstractQueuedSynchronizer {
        /**
         * tryAcquire()最终会调用到此方法上
         * 由于Semaphore是基于资源量的同步器，所以这里的实现会有些不同
         * */
        final int nonfairTryAcquireShared(int acquires) {
            //死循环
            for (;;) {
                //获取当前的资源数量
                int available = getState();
                //减去请求的资源量
                int remaining = available - acquires;
                //如果设置state成功则直接返回剩余资源量
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
}
```

在看了这两个实现后，不知道各位是否对`AQS、ReentrantLock、Semaphore`已经有了一些认识呢，这里简单做个总结——**tryAcquire()**是`AQS`预留的拓展点，并且为实现者提供了一些`cas`方法（如`compareAndSetState()`），实现者根据自己的业务定义是否加锁成功。了解这点后，我们再来看`AQS`加锁的核心方法

##### acquire()

```java
    /**
     * 注意这里是final的，就是不允许子类重写该方法
     * AQS：我自己的实现就够了，没啥问题，你们别再来实现了
     * */
	public final void acquire(int arg) {
        /*
         * tryAcquire()：尝试加锁，如果已经加锁成功，那么这里取反就是false，就不需要再继续往下执行了
         * 如果tryAcquire()加锁失败
         * 那么取反就是true
         * 就要执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)进行排队等待锁
         * */
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //中断当前线程，这里先眼熟一下就好，不用考虑为什么要怎么做
            selfInterrupt();
    }

    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```

假设当前调用了`tryAcquire()`方法，加锁失败，需要进行排队等待锁资源了，首先我们来看`addWaiter(Node.EXCLUSIVE)`方法。注意，这里的`Node.EXCLUSIVE`是标记当前获取锁方式是独占锁

```java
    /**
     * 使用给定模式（这里是独占模式）为当前线程创建结点并排队
     * */
    private Node addWaiter(Node mode) {
        //创建新结点
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // 尝试快速入队，如果失败则进行完全入队
        // 首先获取当前队列的尾结点
        Node pred = tail;
        //如果尾结点不为null
        if (pred != null) {
            //首先将新增结点的前指针指向尾结点
            node.prev = pred;
            //将新增节点通过cas设置到等待队列的尾结点
            if (compareAndSetTail(pred, node)) {
                //将前一个结点的后指针指向新增结点
                pred.next = node;
                return node;
            }
        }
        //如果尾结点是null，则证明队列还没有初始化，进行初始化并入队
        enq(node);
        return node;
    }

	/**
     * 将结点插入队列，必要时对队列进行初始化
     */
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            //如果尾结点是null，需要进行初始化
            if (t == null) {
                /*
                 * 创建一个新结点（注意这里的节点是一个虚结点，什么属性都没有），作为头结点和尾结点
                 * 注意这里的设计思想，头结点只是一个虚结点，用来表示当前锁被占用
                 * */
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                //将新增节点的前一个节点设置为尾结点
                node.prev = t;
                //将新增节点通过cas设置到等待队列的尾结点
                if (compareAndSetTail(t, node)) {
                    //将前一个结点的后指针指向新增结点
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

细心的同学可以发现，在` addWaiter()`和`enq()`方法中有一段同样的代码，如下

```java
private Node addWaiter(Node mode) {
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    } 
}        


private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            //这段代码是完全重复的
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```



对此，作者的解释是：`Try the fast path of enq; backup to full enq on failure`，翻译成中文的意思就是：尝试快速入队，如果失败则进行完全入队。但是实际上，两段代码也只是多了一个`if`判断而已。那为什么会有这种情况呢？这里大胆推测，可能是由于在并发量比较高的情况下，多了一个`if`判断，会有性能上的影响吧，如果有哪位同学了解的话，欢迎赐教。

`addWaiter()`方法介绍完成之后，我们先回顾下`acquire()`方法

```java
public final void acquire(int arg) {
    /*
     * tryAcquire()：尝试加锁，如果已经加锁成功，那么这里取反就是false，就不需要再继续往下执行了
     * 如果tryAcquire()加锁失败
     * 那么取反就是true
     * 就要执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)进行排队等待锁
     * */
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //中断当前线程，这里先眼熟一下就好，不用考虑为什么要怎么做
        selfInterrupt();
}

static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

那么下面就开始分析`acquireQueued()`

##### acquireQueued()

前面分析的`addWaiter()`方法是将当前线程作为一个`node`结点插入到`AQS`的等待队列中，那么`acquireQueued()`说简单点就是处理新插入结点的后续动作，如是否应该自旋获取锁还是阻塞当前线程，代码如下

```java
    final boolean acquireQueued(final Node node, int arg) {
        //标记此次操作是否成功，默认为失败
        boolean failed = true;
        try {
            //当前线程是否被中断
            boolean interrupted = false;
            //自旋
            for (;;) {
                //取出当前节点的前一个结点
                final Node p = node.predecessor();
                /*
                 * 如果前一个节点是头结点，并且当前线程加锁成功
                 * 如果当前判断成立的话，那么代表当前线程已经成功获得了锁， 直接返回即可
                 * */
                if (p == head && tryAcquire(arg)) {
                    //将当前结点设置为头结点
                    setHead(node);
                    p.next = null; // help GC
                    //失败标记置为false
                    failed = false;
                    return interrupted;
                }
                /*
                 * shouldParkAfterFailedAcquire()：获取锁失败后，当前线程是否需要阻塞
                 * parkAndCheckInterrupt()：阻塞当前线程，判断线程是否中断
                 * 注意，这里传入了上一个结点对象
                 * */
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //如果失败，则取消加锁，进行一些后续的清理工作
            if (failed)
                cancelAcquire(node);
        }
    }

    private void setHead(Node node) {
        //将当前结点设置为头结点
        head = node;
        /*
         * 清空变量，这样做有两个好处
         * 1、帮助GC
         * 2、便于其它地方判断（因为我们之前说过头结点只是一个虚结点）
         * */
        node.thread = null;
        node.prev = null;
    }
```

下面再来看看` shouldParkAfterFailedAcquire()`方法是如何判断当前线程是否需要阻塞的

```java
    /**
     * 检查并更新无法获取锁资源的节点的状态，如果线程应该阻塞，则返回true
     * */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //如果上一个结点状态为SIGNAL，当前结点可以直接阻塞
        if (ws == Node.SIGNAL)
            return true;
        //如果大于0，则证明前一个节点为取消状态，遍历删除取消状态的节点
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
            * 如果前一个结点状态是0 or PROPAGATE
            * 那么就使用cas将前一个结点状态设置为SIGNAL，随后返回false，开始下一次自旋
            * */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

如果当前线程需要阻塞，那么`parkAndCheckInterrupt()`会阻塞当前线程，并判断线程是否被中断

```java
    private final boolean parkAndCheckInterrupt() {
        //阻塞当前线程
        LockSupport.park(this);
        //判断当前线程是否被中断
        return Thread.interrupted();
    }
```

还记得`acquire()`中，`if`判断中调用了`selfInterrupt()`方法么，让我们再来回顾一下

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

final boolean acquireQueued(final Node node, int arg) {
    //标记此次操作是否成功，默认为失败
    boolean failed = true;
    try {
        //当前线程是否被中断
        boolean interrupted = false;
        //自旋
        for (;;) {
            //取出当前节点的前一个结点
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; 
                failed = false;
                return interrupted;
            }
            /**
            * shouldParkAfterFailedAcquire()：获取锁失败后，当前线程是否需要阻塞
            * parkAndCheckInterrupt()：阻塞当前线程，判断线程是否中断
            * */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private final boolean parkAndCheckInterrupt() {
    //阻塞当前线程
    LockSupport.park(this);
    //判断当前线程是否被中断
    return Thread.interrupted();
}
```

让我们举例思考一下，假设当前线程`A`已经获取锁了，此时线程`B`开始获取锁，进入到`acquireQueued()`方法后，第一次自旋，竞争锁失败并阻塞在`parkAndCheckInterrupt()`中。当获取锁成功后，其它线程调用`unpark()`方法，此时线程`B`的阻塞解除，并且返回当前线程的中断状态，并在第二次自旋中成功获取到锁，跳出循环。<font color=red>假设在线程`B`阻塞的这段时间，有其它线程调用了线程`B`的`interrupt()`方法，使线程`B`进入到中断状态，那么在线程`B`阻塞解除后，`acquireQueued()`方法返回给`acquire()`的就是`true`，如果是`true`，则在获取到锁资源后中断当前线程。</font>

这几个方法可能比较零散，我们用一张图整体梳理

![AQS加锁流程1](/images/oss/PicGo/AQS%E5%8A%A0%E9%94%81%E6%B5%81%E7%A8%8B1.png)

图中看到，如果线程没有获取到锁，会阻塞住，等待其它线程唤醒，那么这个唤醒操作在哪里呢？接下来我们一起来看`AQS`的解锁操作

#### 解锁

加锁逻辑与加锁类似，同样有两个方法`tryRelease()`和`release()`。`tryRelease()`方法同样需要子类自行实现，这里就不在介绍了，感兴趣的同学可以自行学习。释放锁的逻辑比加锁简单的多，一起来看`release()`方法

##### release()

```java
    public final boolean release(int arg) {
        //如果尝试释放锁成功
        if (tryRelease(arg)) {
            //获取当前头结点
            Node h = head;
            //如果头结点不为空
            if (h != null && h.waitStatus != 0)
                //唤醒该结点的下一个节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

可以看到，释放锁的关键逻辑都在` unparkSuccessor()`方法中

```java
    private void unparkSuccessor(Node node) {
        /*
        * 如果节点状态小于0，则代表处在等待状态，首先清除等待状态
        * */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        //如果当前结点已经被取消了（waitStatus > 0代表该结点已经被取消）
        if (s == null || s.waitStatus > 0) {
            s = null;
            //从后往前找到最前面的未被取消的节点
            //大家可以思考一下，这里为什么要从后面向前找
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //如果不为空则释放锁
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

刚才在代码中，留了一个问题：为什么要从后面向前找？其实这是因为从后向前找可以避免线程并发产生的问题

### 总结

谈到`Java`并发，那`AQS`就是一个绕不开的话题，`Java`中的<font color=red>大部分</font>同步器都是使用`AQS`实现的。此篇文章中介绍了`AQS`的加锁和解锁逻辑和设计思路，与大家共勉。

##### 