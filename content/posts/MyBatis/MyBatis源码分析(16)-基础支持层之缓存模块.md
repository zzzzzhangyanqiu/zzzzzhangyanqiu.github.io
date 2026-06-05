---
title: MyBatis源码分析(16)-基础支持层之缓存模块
tags:
  - MyBatis
date: 2019-01-19
categories:
 - MyBatis
---

## 缓存模块

在各种应用程序中，缓存可以说是提高程序性能非常有效的手段了。`MyBatis`中提供了一级缓存和二级缓存，不过今天我们先不讨论这两级缓存，今天我们来介绍`MyBatis`中的`cache`模块，一级缓存和二级缓存也是通过此模块实现的。`cache`模块使用了**装饰模式**

`MyBatis`的`cache`模块位于`org.apache.ibatis.cache`下，首先看一下此包的结构

![](/images/oss/%E5%8D%9A%E5%AE%A2/org.apache.ibatis.cache%E5%8C%85.png)

首先来看一下`cache`接口

### cache

`Cache`接口做为装饰器中的产品接口（装饰器接口）角色，包含以下方法

```java
public interface Cache {

    /**
   * 获取当前缓存器的标识符
   */
    String getId();

    /**
   * 存入缓存
   */
    void putObject(Object key, Object value);

    /**
   * 从缓存中取出
   */
    Object getObject(Object key);

    /**
   * 移除对应的缓存项
   */
    Object removeObject(Object key);

    /**
   * 清空缓存
   */  
    void clear();

    /**
   * 此方法不再被核心代码调用
   * 获取存储在缓存中的元素数量
   */
    int getSize();

    /** 
   * 截至3.2.6版本  此方法不再被核心代码调用
   *  
   * 缓存所需的任何锁定必须由缓存提供程序在内部提供。
   * 
   * 获取ReadWriteLock
   */
    ReadWriteLock getReadWriteLock();

}
```

`Cache`接口有很多实现类，但除了`PerpetualCache`，其它都是装饰器

![](/images/oss/%E5%8D%9A%E5%AE%A2/cache%E6%8E%A5%E5%8F%A3%E5%AE%9E%E7%8E%B0%E7%B1%BB.png)

### PerpetualCache

`PerpetualCache`在缓存模块中属于被装饰的对象，主要字段如下

```java
/** 当前缓存器的ID **/
private String id;

/** 用来保存缓存的HashMap() **/
private Map<Object, Object> cache = new HashMap<Object, Object>();
```

可以看到，此缓存器实质上就是使用了`HashMap()`来保存数据。而实现的几个方法基本上也都是直接调用`HashMap`的`API`

```java
@Override
public void putObject(Object key, Object value) {
    cache.put(key, value);
}

@Override
public Object getObject(Object key) {
    return cache.get(key);
}

//省略removeObject()等方法。。

/***
   * 之前介绍cache接口的时候说过
   * getReadWriteLock()已经不再被核心代码使用，所以提供空实现
   * **/
@Override
public ReadWriteLock getReadWriteLock() {
    return null;
}

/***
   * 对比缓存器的ID
   * **/
@Override
public boolean equals(Object o) {
    if (getId() == null) {
        throw new CacheException("Cache instances require an ID.");
    }
    if (this == o) {
        return true;
    }
    if (!(o instanceof Cache)) {
        return false;
    }

    Cache otherCache = (Cache) o;
    return getId().equals(otherCache.getId());
}

@Override
public int hashCode() {
    if (getId() == null) {
        throw new CacheException("Cache instances require an ID.");
    }
    return getId().hashCode();
}
```

介绍完了被装饰对象后，下面我们一起来了解一下缓存的装饰器。`cache`模块所有的装饰器都位于`org.apache.ibatis.cache.decorators`包下，这些装饰器会为`PerpetualCache`提供一些额外的功能。

### BlockingCache

`BlockingCache`是一个简单的阻塞装饰器，是`EhCache`的`BlockingCache`装饰器的简单和低效版本。当在缓存中找不到元素时，它会锁定缓存的`key`。 这样，其他线程在访问此`key`时会被阻塞到数据被填充，就不会查询数据库了。首先来看一下主要字段

```java
/** 阻塞的最大时长 **/
private long timeout;

/** 被装饰的缓存器 **/
private final Cache delegate;

/** 存储每一个key对应的ReentrantLock ***/
private final ConcurrentHashMap<Object, ReentrantLock> locks;
```

看到了`locks`，相信很多同学就会反应过来了。原理其实就是在获取缓存数据时，根据`key`申请相应的`ReentrantLock`锁，如果此`key`的锁已经被申请了，则阻塞等待。由于各个方法实现比较类似，我们这里只介绍`getObject()`方法

```java
@Override
public Object getObject(Object key) {
    //根据key申请锁
    acquireLock(key);
    //获取缓存数据
    Object value = delegate.getObject(key);
    if (value != null) {
        //释放锁
        releaseLock(key);
    }        
    return value;
}

private ReentrantLock getLockForKey(Object key) {
    ReentrantLock lock = new ReentrantLock();
    ReentrantLock previous = locks.putIfAbsent(key, lock);
    return previous == null ? lock : previous;
}

/**
   * 申请锁
   * **/
private void acquireLock(Object key) {
    //首先获取此key对应的ReentrantLock对象
    Lock lock = getLockForKey(key);
    //如果超时时间大于0
    if (timeout > 0) {
        try {
            //获取锁，如果已被其它线程获取则阻塞等待
            boolean acquired = lock.tryLock(timeout, TimeUnit.MILLISECONDS);
            if (!acquired) {
                throw new CacheException("...");
            }
        } catch (InterruptedException e) {
            throw new CacheException("...");
        }
    } else {
        //如果超时时间小于0则直接获取锁
        lock.lock();
    }
}

/**
   * 释放锁
   * **/
private void releaseLock(Object key) {
    ReentrantLock lock = locks.get(key);
    //如果是当前线程持有此锁，则释放
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### FifoCache&LruCache

在某些场景中，为了控制缓存的大小，就需要系统按照一定的规则来清理缓存，`FifoCache&LruCache`两个装饰器就是来做这项工作的。

`FifoCache`是根据`Fifo`（先进先出）规则来进行清理的装饰器，该类的主要字段有。

```java
/** 原始缓存器 **/
private final Cache delegate;

/** 保存放入缓存的key值集合 **/
private Deque<Object> keyList;

/** 缓存的大小 **/
private int size;
```

在`FifoCache`的构造方法中，指定了默认的缓存大小

```java
public FifoCache(Cache delegate) {
    this.delegate = delegate;
    //使用LinkedList存储key集合
    this.keyList = new LinkedList<Object>();
    /** 默认缓存数量 **/
    this.size = 1024;
}
```

`FifoCache.putObject()`中调用了清理缓存的逻辑，其它方法均是调用原始缓存器的`API`

```java
@Override
public void putObject(Object key, Object value) {
    //判断并清理缓存
    cycleKeyList(key);
    delegate.putObject(key, value);
}

private void cycleKeyList(Object key) {
    //将此次插入的key存到最后面
    keyList.addLast(key);
    //如果超出大小则清理第一个缓存（也就是最老的一个）
    if (keyList.size() > size) {
        Object oldestKey = keyList.removeFirst();
        delegate.removeObject(oldestKey);
    }
}
```



`LruCache`是根据`LRU`（最近最少使用）规则来清理缓存的装饰器，首先来看一下主要字段

```java
/** 原始缓存器 **/
private final Cache delegate;
/** 使用此集合来存储键值对，
   * 此集合是一个LinkedHashMap集合
   * 实现了removeEldestEntry()方法，稍后会详细说明**/
private Map<Object, Object> keyMap;
/** 最少使用的缓存key **/
private Object eldestKey;
```

`LruCache`的构造方法中会调用`setSize(1024)`来声明默认大小，此方法是实现`LRU`规则的核心。

```java
public void setSize(final int size) {
    //实例化keyMap字段并实现removeEldestEntry()方法
    keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
        private static final long serialVersionUID = 4267176411845948333L;

        @Override
        protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
            //判断当前集合大小是否超出传入的大小
            boolean tooBig = size() > size;
            if (tooBig) {
                //如果已经超出则取出最少使用的key
                eldestKey = eldest.getKey();
            }
            return tooBig;
        }
    };
}
```

这里简单描述一下`removeEldestEntry()`，此方法会在`map.put()/map.putAll()`时被调用，用来判断是否需要删除最少使用的对象，如果该方法返回`true`，则删除最近最少使用的对象，并将删除的对象做为参数传入`removeEldestEntry()`。

`LruCache.putObject()`中会调用`cycleKeyList()`检测是否超出大小，如果超出大小则清理。

```java
@Override
public void putObject(Object key, Object value) {
    delegate.putObject(key, value);
    //检测并清理缓存
    cycleKeyList(key);
}

private void cycleKeyList(Object key) {
    //首先将key存入到keyMap，如果超出大小
    //keyMap.removeEldestEntry()会取出最少使用对象并赋值到eldestKey中
    keyMap.put(key, key);
    //如果不等于空则清理缓存
    if (eldestKey != null) {
        delegate.removeObject(eldestKey);
        eldestKey = null;
    }
}
```

### SoftCache&WeakCache

`SoftCache&WeakCache`两个装饰器分别实现了软引用和虚引用，如果你对`Java`中的强引用、软引用等还不是很熟悉，可以在网上看一些相关的资料。

由于两个装饰器逻辑类似，这里只介绍`SoftCache`，首先来看一下主要字段

```java
/** 将缓存的value加入到此集合以确保不会被GC回受（有强引用） **/
private final Deque<Object> hardLinksToAvoidGarbageCollection;
/** 引用队列，保存已经被GC回收的对象 **/
private final ReferenceQueue<Object> queueOfGarbageCollectedEntries;
/** 原始缓存器 ***/
private final Cache delegate;
/** 最大强引用数量 **/
private int numberOfHardLinks;
```

在`SoftCache`构造方法中初始化了上述几个字段，并且为`numberOfHardLinks`设置了默认值

```java
public SoftCache(Cache delegate) {
    this.delegate = delegate;
    //默认强引用数量为256个
    this.numberOfHardLinks = 256;
    this.hardLinksToAvoidGarbageCollection = new LinkedList<Object>();
    this.queueOfGarbageCollectedEntries = new ReferenceQueue<Object>();
}
```

`SoftCache.putObject()`方法会将`value`包装为`SoftEntry`对象

```java
public void putObject(Object key, Object value) {
    //移除所有的引用队列
    removeGarbageCollectedItems();
    delegate.putObject(key, new SoftEntry(key, value, queueOfGarbageCollectedEntries));
}
```

`SoftEntry`是`SoftCache`的一个内部类，继承自`SoftReference`，`SoftEntry`保证了缓存中的`value`值都是软引用。

```java
private static class SoftEntry extends SoftReference<Object> {
    private final Object key;

    //将value做为软引用，key做为强引用赋值
    SoftEntry(Object key, Object value, ReferenceQueue<Object> garbageCollectionQueue) {
        super(value, garbageCollectionQueue);
        this.key = key;
    }
}
```

`SoftCache.getObject()`方法会从`SoftReference`获取值，并判断是否已经被`GC`，如果已经被`GC`则从缓存中移除。

```java
public Object getObject(Object key) {
    Object result = null;
    //取出指定key对应的软引用
    SoftReference<Object> softReference = (SoftReference<Object>) delegate.getObject(key);
    //如果不是null
    if (softReference != null) {
        //取出value
        result = softReference.get();
        //如果是null证明已经被GC，则从缓存中移除
        if (result == null) {
            delegate.removeObject(key);
        } else {
            // 如果不是空则取出，并添加到
            // hardLinksToAvoidGarbageCollection集合中（做为强引用，防止被回收）
            synchronized (hardLinksToAvoidGarbageCollection) {
                hardLinksToAvoidGarbageCollection.addFirst(result);
                //如果超出最大数量则移除最后一个
                if (hardLinksToAvoidGarbageCollection.size() > numberOfHardLinks) {
                    hardLinksToAvoidGarbageCollection.removeLast();
                }
            }
        }
    }
    return result;
}
```

在调用`SoftCache.putObject()/removeObject()`等方法时，会调用`removeGarbageCollectedItems()`来根据引用队列清理缓存

```java
private void removeGarbageCollectedItems() {
    SoftEntry sv;
    //如果存在于引用队列里面的数据，就证明是被GC了，同时移除缓存里面的数据
    while ((sv = (SoftEntry) queueOfGarbageCollectedEntries.poll()) != null) {
        delegate.removeObject(sv.key);
    }
}
```

### LoggingCache

`LoggingCache`装饰器比较简单，提供了日志记录的功能，会记录缓存的总访问次数和命中的次数

主要字段如下

```java
/** 日志 **/
private Log log;
/** 原始缓存器 **/
private Cache delegate;
/** 总访问次数 **/
protected int requests = 0;
/** 命中次数 **/
protected int hits = 0;
```

`LoggingCache.getObject()`会增加总访问次数和命中次数，并且日志记录

```java
public Object getObject(Object key) {
    //增加总访问次数
    requests++;
    final Object value = delegate.getObject(key);
    //如果命中，则增加命中次数
    if (value != null) {
        hits++;
    }
    //日志记录
    if (log.isDebugEnabled()) {
        log.debug("Cache Hit Ratio [" + getId() + "]: " + getHitRatio());
    }
    return value;
}
```

`LoggingCache.getHitRatio()`用来计算命中的几率

```java
private double getHitRatio() {
    return (double) hits / (double) requests;
}
```

### ScheduledCache

`ScheduledCache`装饰器提供了定时清理缓存的功能，主要字段如下。

```java
/** 原始缓存器 **/
private Cache delegate;
/** 定时清理的周期 **/
protected long clearInterval;
/** 上次清理时间 **/
protected long lastClear;
```

`ScheduledCache`的构造方法中制定了默认的`clearInterval`

```java
public ScheduledCache(Cache delegate) {
    this.delegate = delegate;
    //默认清理周期为1个小时
    this.clearInterval = 60 * 60 * 1000; // 1 hour
    //默认上次清理时间为当前时间
    this.lastClear = System.currentTimeMillis();
}
```

`ScheduledCache`中几乎每个方法都会调用`clearWhenStale`来判断是否应该清理缓存

```java
private boolean clearWhenStale() {
    //如果距离上次清理已经超过了一个小时
    if (System.currentTimeMillis() - lastClear > clearInterval) {
        clear();
        return true;
    }
    return false;
}

```

### TransactionalCache

`TransactionalCache`装饰器提供了事务的功能，是第二级缓存事务缓冲区，此类包含在会话期间要添加到二级缓存的所有缓存对象，调用`commit()`时会将对象保存到缓存，如果回滚，则会将条目丢弃。

该类的大概思路为：在调用`putObject()`方法时，不直接向缓存内保存，而是保存到一个内部的集合中。随后在调用`commit()`方法时，将集合中的对象统一保存到缓存中。

首先来看一下主要字段

```java
/** 日志记录 **/
private static final Log log = LogFactory.getLog(TransactionalCache.class);

/** 原始缓存器 **/
private Cache delegate;

/** 提交事务时是否要清空缓存，此值会在调用clear()时被设置为true
   * 如果此值为true，则不能查询
   * **/
private boolean clearOnCommit;

/** 保存要提交到缓存的集合
   *  做暂存的作用
   * **/
private Map<Object, Object> entriesToAddOnCommit;

/** 保存未命中的key **/
private Set<Object> entriesMissedInCache;
```

`TransactionalCache.putObject()`方法不会直接提交到缓存中，而是向`entriesToAddOnCommit`集合中暂存

```java
public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object);
}
```

`TransactionalCache.commit()/rollback()`分别提供了事务提交和事务回滚的功能。事务提交其实就是把内存中的集合添加到缓存中，而事务回滚则是把集合清空。

```java
public void commit() {
    //根据clearOnCommit值决定是否要清空缓存
    if (clearOnCommit) {
        delegate.clear();
    }
    //将entriesToAddOnCommit集合的值存入缓存
    flushPendingEntries();
    //重置
    reset();
}

public void rollback() {
    //将未命中的key值从缓存中移除
    unlockMissedEntries();
    //重置
    reset();
}

/**
   * 重置
   * **/
private void reset() {
    clearOnCommit = false;
    entriesToAddOnCommit.clear();
    entriesMissedInCache.clear();
}

/**
   * 将集合的中的值存入缓存
   * 未命中的集合也会被存入
   * **/
private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
        delegate.putObject(entry.getKey(), entry.getValue());
    }
    //这里大家可以想一下，为什么要将未命中的集合存入
    for (Object entry : entriesMissedInCache) {
        if (!entriesToAddOnCommit.containsKey(entry)) {
            delegate.putObject(entry, null);
        }
    }
}

/**
   * 将未命中的值从缓存中移除
   * **/
private void unlockMissedEntries() {
    for (Object entry : entriesMissedInCache) {
        try {
            delegate.removeObject(entry);
        } catch (Exception e) {
            log.warn("...");
        }
    }
}
```

`TransactionalCache.clear()`会清空集合，并设置`clearOnCommit`为`true`

```java
public void clear() {
    clearOnCommit = true;
    entriesToAddOnCommit.clear();
}
```

`TransactionalCache.get()`会记录未命中的`key`，并根据`clearOnCommit`的值来决定返回内容

```java
public Object getObject(Object key) {
    // 获取缓存对象
    Object object = delegate.getObject(key);
    if (object == null) {
        //记录未命中
        entriesMissedInCache.add(key);
    }
    // 如果已经调用过了clear()方法，则返回null
    if (clearOnCommit) {
        return null;
    } else {
        return object;
    }
}
```

大家可以想一下，该类为什么不直接加到二级缓存中，而是暂存到了`entriesToAddOnCommit`集合，在`《MyBatis技术内幕》`一书中，徐郡明老师给出了自己的观点。大概意思就是，如果直接存到二级缓存中，会导致脏读的问题，如`事务A`对`数据A`进行了修改和查询操作，`事务B`同时进行查询操作，如果此时`事务A`修改了数据之后直接存到二级缓存中，那`事务B`就会读到未提交的事务，造成脏读。

第二个问题是，为什么在`flushPendingEntries()`方法中，要将`entriesMissedInCache`未命中集合中的内容存入二级缓存中，关于这点，徐郡明老师认为，如果不这样做，并且用户恰好使用了`BlockingCache`装饰器时，由于`BlockingCache.putObject()`会获取锁，`BlockingCache.getObject()`会释放锁，如果不这样做，在两个方法之间发生异常，会导致该缓存一直被锁住，导致其他的`sqlSession`不能读取该缓存。

### SerializedCache&SynchronizedCache

`SerializedCache&SynchronizedCache`两个装饰器实现比较简单，所以放在一起说明。

`SerializedCache`实现了缓存持久化的功能，原理是在`put()`和`get()`方法时，分别使用`ObjectInputStream.writeObject()`和`ObjectInputStream.readObject()`方法来保证持久化。

`SynchronizedCache`实现了同步的功能，原理是在每个方法上加入了`synchronized`关键字。

### CacheKey

使用缓存时一般都需要一个固定的`key`，但是`MyBatis`因为涉及到各种各样的`SQL`语句，如分页参数的不同，用户传入的查询参数不同等等。不能简单的使用`String`对象来做为`key`，所以`MyBatis`中使用了`CacheKey`做为缓存的key值。`CacheKey`对象重写了`equals()`方法，判断两个`key`是否相等，只需要判断`equals()`即可。

首先来看一下主要字段

```java
/** 空的CacheKey **/
public static final CacheKey NULL_CACHE_KEY = new NullCacheKey();

/** multiplier的默认值 **/
private static final int DEFAULT_MULTIPLYER = 37;

/** 默认的hashcode **/
private static final int DEFAULT_HASHCODE = 17;

/** 参与hashcode的计算，默认为37 **/
private int multiplier;

/** hashcode值，默认为17 **/
private int hashcode;

/** 校验值 **/
private long checksum;

/** updateList中的数量 **/
private int count;

/** 由该集合中的所有对象共同计算两个CacheKey是否相等 **/
private List<Object> updateList;
```

在以后的介绍中，可以见到下面四个部分构成的`CacheKey `对象，也就是说这四部分都会记录到该`CacheKey `对象的`updateList `集合中：

1. `MappedStatement `的id 。
2. 指定查询结果集的范围，也就是`RowBounds .offset` 和`RowBounds.limit` 。
3. 查询所使用的SQL 语句，也就是`boundSql.getSql()`方法返回的SQL 语句，其中可能包
   含`？`占位符。
4. 用户传递给上述`SQL `语句的实际参数值。

在向`CacheKey.updateList `添加对象时，调用的是`CacheKey.update()`方法

```java
public void update(Object object) {
    //如果是集合则循环添加
    if (object != null && object.getClass().isArray()) {
        int length = Array.getLength(object);
        for (int i = 0; i < length; i++) {
            Object element = Array.get(object, i);
            doUpdate(element);
        }
    } else {
        //每次添加一个
        doUpdate(object);
    }
}

private void doUpdate(Object object) {
    //传入参数是否等于空
    int baseHashCode = object == null ? 1 : object.hashCode();

    //重新计算计算count和checksum
    count++;
    checksum += baseHashCode;
    baseHashCode *= count;

    //重新计算hashcode
    hashcode = multiplier * hashcode + baseHashCode;

    //增加到updateList集合中
    updateList.add(object);
}
```

`CacheKey `还提供了`doUpdate()`方法对数组批量添加

```java
public void updateAll(Object[] objects) {
    for (Object o : objects) {
        update(o);
    }
}
```

`CacheKey`重写了`equals()/hashCode()`两个方法来判断两个对象是否相等

```java
@Override
public boolean equals(Object object) {
    if (this == object) {
        return true;
    }

    if (!(object instanceof CacheKey)) {
        return false;
    }

    //转换为CacheKey对象
    final CacheKey cacheKey = (CacheKey) object;

    //判断hashcode、checksum、count等字段
    if (hashcode != cacheKey.hashcode) {
        return false;
    }
    if (checksum != cacheKey.checksum) {
        return false;
    }
    if (count != cacheKey.count) {
        return false;
    }

    //循环判断updateList中的对象
    for (int i = 0; i < updateList.size(); i++) {
        Object thisObject = updateList.get(i);
        Object thatObject = cacheKey.updateList.get(i);
        if (thisObject == null) {
            if (thatObject != null) {
                return false;
            }
        } else {
            if (!thisObject.equals(thatObject)) {
                return false;
            }
        }
    }
    return true;
}

@Override
public int hashCode() {
    //直接返回hashcode字段的值
    return hashcode;
}
```









































