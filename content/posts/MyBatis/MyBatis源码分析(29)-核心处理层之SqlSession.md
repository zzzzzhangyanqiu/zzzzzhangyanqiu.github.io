---
title: MyBatis源码分析(29)-核心处理层之SqlSession
tags:
  - MyBatis
date: 2019-05-06
categories:
 - MyBatis
---

### SqlSession

说起`SqlSession`，大家肯定都不陌生，在编写官网`demo`时候就是使用的`SqlSession`完成数据库的操作，对于`MyBatis`来说，`SqlSession`也是对外供应用程序调用的唯一接口。首先看一下此模块的类图

![](/images/oss/%E5%8D%9A%E5%AE%A2/MyBatis/sqlsession%E7%B1%BB%E5%9B%BE.png)

整个模块的实际比较简单，`SqlSessionFactory/DefaultSqlSessionFactory`负责创建`SqlSession`，`SqlSession/DefaultSqlSession`负责调用`Executor`执行数据库操作。唯一比较特殊的是`SqlSessionManager`类，类图中可以看出，它继承了两个接口，所以，它既可以创建`SqlSession`，也可以调用`Executor`，并且可以根据线程复用`SqlSession`，后面会进行详细分析

首先来看`SqlSessionFactory`

#### SqlSessionFactory

为了方便使用，`SqlSessionFactory`中提供了不同方式的重载，如下

```java
/***
   * 创建SqlSession对象
   * **/
SqlSession openSession();

SqlSession openSession(boolean autoCommit);
SqlSession openSession(Connection connection);
SqlSession openSession(TransactionIsolationLevel level);

SqlSession openSession(ExecutorType execType);
SqlSession openSession(ExecutorType execType, boolean autoCommit);
SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);
SqlSession openSession(ExecutorType execType, Connection connection);

/***
   * 获取全局Configuration
   * **/
Configuration getConfiguration();
```

在`DefaultSqlSessionFactory`的实现中，所有重载最终都会调用到两个方法上，分别是`openSessionFromDataSource()/openSessionFromConnection()`，两个方法逻辑相同且实现比较简单，大家可自行分析。

#### SqlSession

`SqlSession`作为`MyBatis`的顶级接口，提供了所有操作数据库的方法，如下

```java
/**
   * 查询单个对象，参数为SQL语句
   */
<T> T selectOne(String statement);

/**
   * 查询单个对象，参数为SQL语句和SQL语句中的实参
   */
<T> T selectOne(String statement, Object parameter);

/**
   * 查询集合
   */
<E> List<E> selectList(String statement);

<E> List<E> selectList(String statement, Object parameter);

/**
   * 查询集合，第三个参数为行数限制
   */
<E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);

/**
   * 查询。根据生成的对象中的一个属性将结果列表转换为Map
   */
<K, V> Map<K, V> selectMap(String statement, String mapKey);

<K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey);

<K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds);

/**
   * 查询并返回游标对象
   */
<T> Cursor<T> selectCursor(String statement);

<T> Cursor<T> selectCursor(String statement, Object parameter);

<T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds);

/**
   * 查询
   */
void select(String statement, Object parameter, ResultHandler handler);

void select(String statement, ResultHandler handler);

void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler);

/**
   * 插入
   */
int insert(String statement);

int insert(String statement, Object parameter);

/**
   * 更新
   */
int update(String statement);

int update(String statement, Object parameter);

/**
   * 删除
   */
int delete(String statement);

/**
   * 删除
   */
int delete(String statement, Object parameter);

/**
   * 刷新批处理语句并提交事务
   */
void commit();

void commit(boolean force);

/**
   * 丢弃待处理的批处理语句并回滚事务
   */
void rollback();

void rollback(boolean force);

/**
   * 刷新批处理语句，也就是批量执行
   */
List<BatchResult> flushStatements();

/**
   * 关闭会话
   */
@Override
void close();

/**
   * 清除本地缓存
   */
void clearCache();

/**
   * 获取当前配置
   */
Configuration getConfiguration();

/**
   * 根据类型返回mapper
   */
<T> T getMapper(Class<T> type);

/**
   * 获取数据库连接
   */
Connection getConnection();
```

`DefaultSqlSession`是最常见的`SqlSession`接口的实现类，字段如下

```java
/**
   * 配置
   * **/
private Configuration configuration;
/**
   * 当前对应的执行器
   * **/
private Executor executor;

/**
   * 是否自动提交
   * **/
private boolean autoCommit;
/**
   * 是否存在脏数据
   * **/
private boolean dirty;
/**
   * 由于Cursor使用后需要关闭，MyBatis为了防止外部程序忘记关闭
   * 会将调用的Cursor存储在此，在close时统一关闭
   * **/
private List<Cursor<?>> cursorList;
```

在`SqlSession`接口中，我们看到了很多方法，其实很多方法最终调用的都是同一个方法，如`selectOne()/selectMap()/selectList()/select()`调用的都是`DefaultSqlSession.selectList()`，再由`selectList()`方法调用`Executor.query()`，最后各自处理返回值。

`insert()/update()/delete()`最终调用的都是`DefaultSqlSession.update()`，而该方法会调用`Executor.update()`，以`selectOne()`方法举例

```java
public <T> T selectOne(String statement, Object parameter) {
    //该方法会调用executor.query()进行查询
    List<T> list = this.<T>selectList(statement, parameter);
    if (list.size() == 1) {
        return list.get(0);
    } else if (list.size() > 1) {
        //如果结果大于1则抛出异常
        throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
        //如果结果为0则返回null
        return null;
    }
}
```

`DefaultSqlSession.update()`方法会将`dirty`属性设置为`true`，并在`commit()/callback()/close()`方法中重设为`false`

其余方法实现都比较简单，各位可自行分析

#### SqlSessionManager

前面说到，`SqlSessionManager`比较特殊，因为此类继承了两个接口，所以既可以创建`SqlSession`，也可以调用`Executor`，首先来看其字段

```java
/**
   * 引用的SqlSessionFactory和SqlSession代理类
   * **/
private final SqlSessionFactory sqlSessionFactory;
private final SqlSession sqlSessionProxy;

/***
   * 为每个线程存储SqlSession
   * 该集合可以保证每个线程复用一个SqlSession对象
   * **/
private ThreadLocal<SqlSession> localSqlSession = new ThreadLocal<SqlSession>();
```

在`SqlSessionManager`的构造方法中，会为`SqlSession`创建代理类

```java
private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
    this.sqlSessionFactory = sqlSessionFactory;
    //创建代理类
    this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[]{SqlSession.class},
        new SqlSessionInterceptor());
}
```

`SqlSessionInterceptor`是`SqlSessionManager`的内部类，既然实现了动态代理，那么我们肯定要关注`invoke()`方法

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //获取当前线程的SqlSession对象
    final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
    if (sqlSession != null) {
        //如果获取到，直接使用
        try {
            return method.invoke(sqlSession, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
    } else {
        //如果没有获取到则新建
        final SqlSession autoSqlSession = openSession();
        try {
            final Object result = method.invoke(autoSqlSession, args);
            autoSqlSession.commit();
            return result;
        } catch (Throwable t) {
            //如果发生异常则回滚
            autoSqlSession.rollback();
            throw ExceptionUtil.unwrapThrowable(t);
        } finally {
            autoSqlSession.close();
        }
    }
}
```

`SqlSessionManager`的其它方法如`openSession()/select()/commit()`等都是通过调用两个字段的相关方法实现的，这里就不在分析了，最后提一句，由于现在使用`MyBatis`都是与`Spring`进行集成，其中会用到另外一个实现类`SqlSessionTemplate`（以后进行分析），所以`SqlSessionManager`用的已经很少了。