---
title: MyBatis源码分析(28)-核心处理层之Executor
tags:
  - MyBatis
categories:
 - MyBatis
date: 2019-05-04
---

### Executor

`Executor`是`MyBatis`中比较核心的模块之一，该模块负责调用之前分析过的`StatementHandler`来完成一系列对数据库的操作。首先看一下此模块中一些核心类的类图

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/MyBatis/Executor%E7%B1%BB%E5%9B%BE.png)

在整个模块的结构中，`MyBatis`用到了模板模式和装饰模式，对于这两种设计模式还不熟悉的同学，可以先看一下[这篇文章](https://suiyueranzly.gitee.io/posts/2130093882/)或者去网上找一些其他的资料。`MyBatis`的大致设计思路是，在`CachingExecutor`中使用装饰模式来为其他的`Executor`提供缓存功能，在`BaseExecutor`中使用模板模式来为其三个子类提供一些公共的方法。`CachingExecutor`及二级缓存后面会进行详细分析。了解了大概结构后，首先先来分析`Executor`接口，该接口主要方法如下

```java
/**
   * 执行数据库更新操作
   * **/
int update(MappedStatement ms, Object parameter) throws SQLException;

/**
   * 查询并返回list集合
   * **/
<E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;

<E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;

/**
   * 查询并返回游标对象
   * **/
<E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;

/**
   * 批处理
   * **/
List<BatchResult> flushStatements() throws SQLException;

/**
   * 事务提交
   * **/
void commit(boolean required) throws SQLException;

/**
   * 事务回滚
   * **/
void rollback(boolean required) throws SQLException;

/**
   * 创建缓存需要的CacheKey对象
   * **/
CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);

/**
   * 查询缓存中是否包含指定对象
   * **/
boolean isCached(MappedStatement ms, CacheKey key);

/**
   * 清除一级缓存
   * **/
void clearLocalCache();

/**
   * 嵌套查询的延迟加载
   * **/
void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);

/**
   * 获取事务
   * **/
Transaction getTransaction();

/**
   * 关闭此执行器
   * **/
void close(boolean forceRollback);

/**
   * 判断是否关闭
   * **/
boolean isClosed();

/**
   * 创建执行器的装饰器
   * **/
void setExecutorWrapper(Executor executor);
```

#### BaseExecutor

前面提到过，`BaseExecutor`使用了模板方法为其子类提供了一些`公共操作`，这里的`公共操作`主要指的是两个功能：**一级缓存/事务管理**，子类只需要关注基本的增删改查的实现即可，需要子类关注的主要有四个方法，分别是`doUpdate()/doFlushStatements()/doQuery()/doQueryCursor()`稍后会依次进行分析，首先来了解一下`MyBatis`中的**一级缓存**

##### 一级缓存

在执行查询时，有的时候会遇到查询的`SQL`语句一样的情况，`MyBatis`针对这种情况做了一级缓存优化，如果执行相同的`SQL`查询，则第二次会直接从缓存中取出查询结果，减少了数据库的开销，一级缓存是`SqlSession`级别的，`SqlSession`关闭后，对应的一级缓存也会被销毁，而`Executor`才是真正管理一级缓存的对象，如果`Executor`被关闭，其持有的一级缓存也会失效。

简单的介绍了一级缓存后，继续回到`BaseExecutor`，该类的字段如下

```java
/***
   * 事务对象
   * **/
protected Transaction transaction;
/***
   * 当前执行的装饰器
   * **/
protected Executor wrapper;

/**
   * 延迟加载队列
   * **/
protected ConcurrentLinkedQueue<DeferredLoad> deferredLoads;
/**
   * 对于查询结果的一级缓存，PerpetualCache前面的文章分析过
   * **/
protected PerpetualCache localCache;
/**
   * 对于输出型参数的一级缓存
   * ***/
protected PerpetualCache localOutputParameterCache;
protected Configuration configuration;

/**
   * 嵌套层数
   * **/
protected int queryStack;
/***
   * 当前执行器是否关闭
   * **/
private boolean closed;
```

首先来看`BaseExecutor.query()`方法，该方法会首先创建`CacheKey`对象，并查询一级缓存中是否存在，如果存在则直接返回，不存在则调用子类的查询逻辑

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //获取BoundSql对象
    BoundSql boundSql = ms.getBoundSql(parameter);
    //创建cacheKey
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    //重载
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}


public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    //判断是否关闭
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    //如果嵌套次数为0，并且配置了FlushCacheRequired为true。则清空一级缓存
    //clearLocalCache()会调用localCache/localOutputParameterCache两个集合的clear()方法
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        //嵌套查询次数+1
        queryStack++;
        //查询缓存中是否存在
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            //处理输出型参数的缓存
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            //如果缓存中不存在则向数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        //嵌套查询次数-1
        queryStack--;
    }
    //如果嵌套查询此书为0，则开始处理deferredLoads对象
    //deferredLoads里面存储了需要延迟加载的对象，稍后会进行分析
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        //清除集合
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // 根据LocalCacheScope的配置决定是否清除缓存
            clearLocalCache();
        }
    }
    return list;
}
```

整个`query()`的调用逻辑如上所示，下面我们分析几个比较重要的点，

`createCacheKey()`方法会为每一次查询创建`cacheKey`对象用来查询缓存，如下

```java
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    //判断是否关闭
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    //首先加入MappedStatement的ID
    cacheKey.update(ms.getId());
    //offset和limit
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    //本次查询的SQL语句
    cacheKey.update(boundSql.getSql());
    //用户传入的实参
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    for (ParameterMapping parameterMapping : parameterMappings) {
        if (parameterMapping.getMode() != ParameterMode.OUT) {
            Object value;
            String propertyName = parameterMapping.getProperty();
            if (boundSql.hasAdditionalParameter(propertyName)) {
                value = boundSql.getAdditionalParameter(propertyName);
            } else if (parameterObject == null) {
                value = null;
            } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                value = parameterObject;
            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                value = metaObject.getValue(propertyName);
            }
            cacheKey.update(value);
        }
    }
    //environmentId
    if (configuration.getEnvironment() != null) {
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}
```

根据上述方法可以看出，一次`cacheKey`对象由多个元素组成，这样才能确保是同一次查询。

`query()`方法中的`queryFromDatabase()`会直接调用子类的查询逻辑

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    //在localCache中添加占位符，
    // 稍后会删除此占位符，
    // 可以避免两个查询后发现缓存中已经存在的情况
    // 也可以标记此查询是否已经完成
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        //doQuery()方法是一个abstract方法，交给子类去自行实现
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        //查询结束后移除一级缓存中的占位符
        localCache.removeObject(key);
    }
    //将查询结果放入缓存
    localCache.putObject(key, list);
    //如果statementType是callable类型，还要处理输出类型参数的缓存
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

`query()`方法的逻辑这里就结束了，下面分析一下嵌套查询的延迟加载，也就是`DeferredLoad`

##### DeferredLoad

`DeferredLoad`是`BaseExecutor`的一个内部类。其主要字段如下

```java
/***
         * 嵌套查询外层结果对象对应的MetaObject
         * **/
private final MetaObject resultObject;
/**
         * 该结果在外层对象中的字段名，用来赋值
         * ***/
private final String property;
/**
         * 该结果的类型
         * ***/
private final Class<?> targetType;
/***
         * 该结果的CacheKey
         * **/
private final CacheKey key;
/**
         * 与BaseExecutor.localCache保持一致
         * **/
private final PerpetualCache localCache;
/**
         * 用于创建类
         * ***/
private final ObjectFactory objectFactory;
/**
         * 此对象用于将List转换为Object，前面已经分析过了
         * ***/
private final ResultExtractor resultExtractor;
```

`DeferredLoad`只有两个方法，`canLoad()`用来判断是否可以进行加载，`Load()`用来进行加载

```java
public boolean canLoad() {
    //是否可以进行加载的条件有两个
    //1、一级缓存中存在此key的数据
    //2、一级缓存中存在的数据不是占位符，也就意味着查询操作已经执行完了
    return localCache.getObject(key) != null && localCache.getObject(key) != EXECUTION_PLACEHOLDER;
}
```

之前分析`DefaultResultSetHandler.getNestedQueryMappingValue()`方法时，有一段与嵌套查询延迟加载有关的代码，如下

```java
//如果该嵌套查询已经执行过
if (executor.isCached(nestedQuery, key)) {
    //创建deferLoad对象，并通过deferLoad对象直接从缓存中加载结果
    //关于executor和deferLoad相关以后进行分析
    executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
    //返回特殊标记
    value = DEFERED;
} 
```

`executor.deferLoad()`会判断当前是否可以加载，如果可以则进行加载

```java
public void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType) {
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    //创建DeferredLoad对象
    DeferredLoad deferredLoad = new DeferredLoad(resultObject, property, key, localCache, configuration, targetType);
    //如果可以加载，则直接进行加载，否则添加到deferredLoads队列中
    if (deferredLoad.canLoad()) {
        deferredLoad.load();
    } else {
        deferredLoads.add(new DeferredLoad(resultObject, property, key, localCache, configuration, targetType));
    }
}
```

接下来开始分析`BaseExecutor`的三个子类

##### SimpleExecutor

`SimpleExecutor`是最简单的一个子类，其`doUpdate()/doQuery()/doQueryCursor()`都是直接调用`StatementHandler`的响应方法实现，这里就不在分析了，`SimpleExecutor.prepareStatement()`用来创建`Statement`对象

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    //创建Statement并调用parameterize完成SQL语句参数的赋值
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
}
```

另外，由于该类不支持批量操作，`doFlushStatements()`会直接返回空集合

```java
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    return Collections.emptyList();
}
```

##### ReuseExecutor

我们知道，执行`SQL`语句的时候都要通过`Statement`对象，每次执行都需要重新创建销毁`Statement`，比较耗费资源，针对这种情况，`MyBatis`提供了缓存`Statement`对象的机制，避免重复创建，`ReuseExecutor`的特点就是可重用`Statement`，其实现原理是通过内部的一个字段`private final Map<String, Statement> statementMap = new HashMap<String, Statement>();`，该字段`key`为操作的`SQL`，`value`为`Statement`对象。`ReuseExecutor`的`doUpdate()/doQuery()/doQueryCursor()`与`SimpleExecutor`实现逻辑类似，可自行分析，主要在于`prepareStatement()`方法

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    //获取SQL
    BoundSql boundSql = handler.getBoundSql();
    String sql = boundSql.getSql();
    //如果此SQL语句在缓存中已经有了Statement对象，则直接重用
    if (hasStatementFor(sql)) {
        stmt = getStatement(sql);
        //设置超时时间等
        applyTransactionTimeout(stmt);
    } else {
        //如果没有则新建后加入缓存
        Connection connection = getConnection(statementLog);
        stmt = handler.prepare(connection, transaction.getTimeout());
        //添加到statementMap缓存中
        putStatement(sql, stmt);
    }
    //处理参数
    handler.parameterize(stmt);
    return stmt;
}
```

`hasStatementFor()`会查询缓存中是否有对应的`Statement`对象

```java
private boolean hasStatementFor(String sql) {
    try {
        //1、statementMap中有此SQL对应的statement
        //2、对应的statement没有被关闭
        return statementMap.keySet().contains(sql) && !statementMap.get(sql).getConnection().isClosed();
    } catch (SQLException e) {
        return false;
    }
}
```

由于在调用事务`commit()/rollback()`等方法时都会调用`doFlushStatements()`进行批处理，所以在`ReuseExecutor.doFlushStatements()`中，会清空`statementMap`集合并销毁所有的`Statement`对象

```java
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    for (Statement stmt : statementMap.values()) {
        //该方法会调用statement.close()进行关闭
        closeStatement(stmt);
    }
    statementMap.clear();
    //不支持批量操作，所以返回空集合
    return Collections.emptyList();
}
```

重用`Statement`虽然节省了开销，但是也带来了一些问题，如查询`Cursor`对象时，通过前面的分析我们知道，只有调用`Cursor.next()`的时候，才会去映射此条记录，但是一个`Statement`只能对应一个`cursor`对象，并且调用`DefaultCursor，next()`方法时，会关闭`statement`，这样就导致了一个问题，如果使用同样的`Statement`对象查询两个`cursor`，那么第一个就会失效。

##### BatchExecutor

程序使用过程中，如果执行的`SQL`较多，频繁的打开关闭数据库必定会耗费网络资源，针对这种情况，`MyBatis`提供了批处理的机制，多个操作只会打开一次数据库，节省资源消耗，需要注意的是，数据库对于单次连接执行的操作数也是有限制的，如果超出了这个限制，则会异常。`BatchExecutor`的特点就是可以进行批量操作，主要实现思路是在`doUpdate()`方法中，不直接进行数据库的操作，会将同一类型的`SQL`调用`StatementHandler.batch()`存储到同一个`Statement`对象中，在`doFlushStatements()`方法中再进行批量执行，其主要字段如下

```java
/***
     * 存储了批量执行的Statement对象
     * **/
private final List<Statement> statementList = new ArrayList<Statement>();
/**
     * 存储批量执行的结果
     * **/
private final List<BatchResult> batchResultList = new ArrayList<BatchResult>();
/**
     * 存储当前正在执行的SQL
     * ***/
private String currentSql;
/**
     * 当前的MappedStatement对象
     * **/
private MappedStatement currentStatement;
```

首先简单来看一下`BatchResult`类

###### BatchResult

该类中只有几个字段和`get()/set()`方法，其字段如下

```java
/**
   * 保存当前的MappedStatement
   * **/
private final MappedStatement mappedStatement;
/**
   * 保存当前的sql语句
   * **/
private final String sql;
/**
   * 保存当前用户传入的实参
   * **/
private final List<Object> parameterObjects;
/**
   * SQL语句执行完成后受影响的行数
   * 由于是批量执行，所以这里是数组
   * **/
private int[] updateCounts;
```

继续回到`BatchExecutor`，`BatchExecutor`的`doQuery()/doQueryCursor()`比较简单，唯一需要注意的是，这两个方法开始都会调用`flushStatements()`进行批量执行（进行批量执行再查询，这样才能保证得到最新的查询结果），主要分析`doUpdate()`方法

```java
public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    //获得StatementHandler对象
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    //获得SQL语句
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;
    //如果SQL语句相同并且MappedStatement相同，则可以证明是统一类型的语句
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
        //去除集合中的最后一个Statement重用
        int last = statementList.size() - 1;
        stmt = statementList.get(last);
        //设置超时时间，处理参数等。。
        applyTransactionTimeout(stmt);
        handler.parameterize(stmt);
        //将用户传入的实参加入到最后一个BatchResult钟
        BatchResult batchResult = batchResultList.get(last);
        batchResult.addParameterObject(parameterObject);
    } else {
        //如果SQL和当前执行的不同，则新建Statement对象并添加到statementList中
        Connection connection = getConnection(ms.getStatementLog());
        stmt = handler.prepare(connection, transaction.getTimeout());
        handler.parameterize(stmt);    //fix Issues 322
        currentSql = sql;
        currentStatement = ms;
        statementList.add(stmt);
        batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    //不知道各位是否还有印象，此方法会调用Statement.batch()
    handler.batch(stmt);
    return BATCH_UPDATE_RETURN_VALUE;
}
```

在`BatchExecutor.doFlushStatements()`方法中，会调用`statement.executeBatch()`将记录的`SQL`全部执行

```java
public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
        //用来存储结果
        List<BatchResult> results = new ArrayList<BatchResult>();
        //isRollback参数含义为是否回滚事务，true为回滚，false为不回滚
        //如果回滚则直接返回空集合
        if (isRollback) {
            return Collections.emptyList();
        }
        //遍历statementList
        for (int i = 0, n = statementList.size(); i < n; i++) {
            Statement stmt = statementList.get(i);
            //设置超时时间
            applyTransactionTimeout(stmt);
            //获取对应BatchResult
            BatchResult batchResult = batchResultList.get(i);
            try {
                //调用Statement.executeBatch()批量执行
                //并将结果记录到BatchResult.updateCounts中
                batchResult.setUpdateCounts(stmt.executeBatch());
                //获取当前的MappedStatement和用户传入实参，主要是为了对主键赋值
                MappedStatement ms = batchResult.getMappedStatement();
                List<Object> parameterObjects = batchResult.getParameterObjects();
                KeyGenerator keyGenerator = ms.getKeyGenerator();
                if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
                    //如果是Jdbc3KeyGenerator类型，则执行processBatch()方法进行后续处理
                    Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
                    jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
                } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) {
                    //如果是其他类型则调用processAfter()
                    for (Object parameter : parameterObjects) {
                        keyGenerator.processAfter(this, ms, stmt, parameter);
                    }
                }
            } catch (BatchUpdateException e) {
                //抛出异常
                StringBuilder message = new StringBuilder();
                message.append(batchResult.getMappedStatement().getId())
                    .append(" (batch index #")
                    .append(i + 1)
                    .append(")")
                    .append(" failed.");
                if (i > 0) {
                    message.append(" ")
                        .append(i)
                        .append(" prior sub executor(s) completed successfully, but will be rolled back.");
                }
                throw new BatchExecutorException(message.toString(), e, results, batchResult);
            }
            //存储执行结果
            results.add(batchResult);
        }
        return results;
    } finally {
        for (Statement stmt : statementList) {
            closeStatement(stmt);
        }
        currentSql = null;
        statementList.clear();
        batchResultList.clear();
    }
}
```

#### CachingExecutor

`CachingExecutor`是一个装饰器，主要是为其他的`Executor`赋予二级缓存的作用，这里简单分析一下二级缓存。

##### 二级缓存

通过前面的分析我们知道，`MyBatis`的一级缓存是会话级别的，如果此次会话被关闭，则缓存也会失效，而二级缓存是整个应用级别的，只要应用不关闭，缓存就会一直有效。关于二级缓存，主要有以下几个配置影响

1、`mybatis-config.xml`中，`setting`标签中的`cacheEnabled`配置项，该配置项会全局地开启或关闭配置文件中的所有映射器已经配置的任何缓存，默认为`true`

2、  `mapper.xml`中的`cache`或`cache-ref`标签。只要该配置文件中出现了该标签，则证明此`mapper`开启了二级缓存，需要注意的是，如果`mapper.xml`文件中配置了`cache-ref`标签，则此`mapper`不会有单独的二级缓存对象，而是与配置共享一个二级缓存对象

3、`select`标签的`useCache`属性，将其设置为 `true `后，将会导致本条语句的结果被二级缓存缓存起来，默认值：对 `select `元素为 `true`。

`CachingExecutor`实现二级缓存主要依赖于`TransactionalCache`和`TransactionalCacheManager`两个类，关于`TransactionalCache`类，前面分析过，大家可以看[这篇文章](https://suiyueranzly.gitee.io/posts/44241008/)做一个简单的回顾，这里简单分析一下`TransactionalCacheManager`，该类内部以一个`map`集合来维护二级缓存的对应关系，如下

```java
/**
   * 其中key为缓存对象，value为对应的TransactionalCache
   * **/
private Map<Cache, TransactionalCache> transactionalCaches = new HashMap<Cache, TransactionalCache>();
```

`TransactionalCacheManager`的方法及其简单，都是调用`TransactionalCache`的相应方法实现的，以`commit()`为例，其余方法可自行分析

```java
public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
        txCache.commit();
    }
}
```

分析过二级缓存的相关知识后，一起来看`CachingExecutor`的实现，`CachingExecutor`中只有两个字段，如下

```java
/**
   * 原始执行器
   * **/
private Executor delegate;
/**
   * 事务缓存管理器
   * **/
private TransactionalCacheManager tcm = new TransactionalCacheManager();
```

在该类的构造方法中，会将自己作为装饰器来装饰原始执行器

```java
public CachingExecutor(Executor delegate) {
    this.delegate = delegate;
    delegate.setExecutorWrapper(this);
}
```

既然涉及到缓存，那么我们主要关注其`query()`方法即可

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    //获取BoundSql对象
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    //获取cacheKey
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    //调用重载
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

`query()`重载方法会首先向缓存中获取数据，如果不存在才会查询数据库

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
    //获取当前mapper的缓存对象
    Cache cache = ms.getCache();
    //如果开启了二级缓存
    if (cache != null) {
        //根据flushCache的配置决定是否要清空缓存
        flushCacheIfRequired(ms);
        //如果使用了二级缓存
        if (ms.isUseCache() && resultHandler == null) {
            //确保没有输出类型的参数
            ensureNoOutParams(ms, parameterObject, boundSql);
            //从缓存中获取
            List<E> list = (List<E>) tcm.getObject(cache, key);
            //如果没有，则进行数据库查询后放入缓存
            if (list == null) {
                list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                tcm.putObject(cache, key, list); 
            }
            return list;
        }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

关于执行器的核心内容到这里就结束了