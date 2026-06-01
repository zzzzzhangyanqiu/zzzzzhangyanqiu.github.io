---
title: MyBatis源码分析(13)-基础支持层之数据源
tags:
  - MyBatis
date: 2018-12-09
categories:
 - MyBatis
---

## 数据源

在数据持久层中，数据源是一个非常重要的组件，其性能直接关系到整个数据持久层的性能。在实践中比较常见的第三方数据源组件有`Apache Common DBCP`,`C3PO` 、`Proxool `等，` MyBatis`不仅可以集成第三方数据源组件，还提供了自己的数据源实现。` MyBatis`的数据源模块是工厂模式的一个典型应用，我们可以从中借鉴其思想，如果你还不是很熟悉，建议看一下[这篇文章](https://suiyueranzly.gitee.io/posts/3439274786/)或者去网上找一些相关资料。

常见的数据源组件都实现了`javax.sql.DataSource`接口，`MyBatis`中提供了两个该接口的实现类，分别是`PooledDataSource`，`UnpooledDataSource`，首先来看一下整体架构图

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/%E6%95%B0%E6%8D%AE%E6%BA%90%E6%A8%A1%E5%9D%97%E7%B1%BB%E5%9B%BE.png)

首先来看一下`DataSourceFactory`

### DataSourceFactory

`DataSourceFactory`做为工厂模式中的抽象工厂角色，提供了两个方法

```java
/**
   * 设置属性
   * **/
void setProperties(Properties props);

/**
   * 获取DataSource对象
   * **/
DataSource getDataSource();
```

`DataSourceFactory`有两个实现类，我们这里只分析`UnpooledDataSourceFactory`，有兴趣的同学可以自行学习`JndiDataSourceFactory`

### 非池连接

“非池连接”的意思为只是提供了数据库连接，但是并没有使用池化技术。

#### UnpooledDataSourceFactory

通过名字我们可以看出来，此`DataSourceFactory`并没有使用池化技术，首先看一下该类的属性

```java
  /**
   * DataSource配置项的前缀
   * **/
private static final String DRIVER_PROPERTY_PREFIX = "driver.";
private static final int DRIVER_PROPERTY_PREFIX_LENGTH = DRIVER_PROPERTY_PREFIX.length();

  /**
   * DataSource对象，该属性会在类初始化时被实例化为UnpooledDataSource对象
   * **/
protected DataSource dataSource;

public UnpooledDataSourceFactory() {
    this.dataSource = new UnpooledDataSource();
}
```

`getDataSource()`方法比较简单，直接返回`dataSource`属性，`setProperties()`方法中提供了配置数据源的功能

```java
@Override
public DataSource getDataSource() {
    return dataSource;
}

@Override
public void setProperties(Properties properties) {
    //保存数据源的配置
    Properties driverProperties = new Properties();
    //为dataSource属性创建MetaObject对象，
    //稍后使用此对象来对dataSource中的driverProperties赋值
    //从而实现保存数据源配置的目的
    MetaObject metaDataSource = SystemMetaObject.forObject(dataSource);
    for (Object key : properties.keySet()) {
        String propertyName = (String) key;
        //如果配置项是以"driver."开头，那么证明是数据源的配置
        if (propertyName.startsWith(DRIVER_PROPERTY_PREFIX)) {
            String value = properties.getProperty(propertyName);
            driverProperties.setProperty(propertyName.substring(DRIVER_PROPERTY_PREFIX_LENGTH), value);
        } else if (metaDataSource.hasSetter(propertyName)) {
            //如果不是以"driver."开头则判断dataSource中是否有此属性的setter方法，如果有则直接赋值
            String value = (String) properties.get(propertyName);
            Object convertedValue = convertValue(metaDataSource, propertyName, value);
            metaDataSource.setValue(propertyName, convertedValue);
        } else {
            throw new DataSourceException("....“);
        }
    }
    if (driverProperties.size() > 0) {
        metaDataSource.setValue("driverProperties", driverProperties);
    }
}
                                          
private Object convertValue(MetaObject metaDataSource, String propertyName, String value) {
    Object convertedValue = value;
    //取出该属性setter方法的参数类型进行强转
    Class<?> targetType = metaDataSource.getSetterType(propertyName);
    if (targetType == Integer.class || targetType == int.class) {
      convertedValue = Integer.valueOf(value);
    } else if (targetType == Long.class || targetType == long.class) {
      convertedValue = Long.valueOf(value);
    } else if (targetType == Boolean.class || targetType == boolean.class) {
      convertedValue = Boolean.valueOf(value);
    }
    return convertedValue;
  }
```

#### UnpooledDataSource

`UnpooledDataSourceFactory`主要负责创建`UnpooledDataSource`，`javax.sql.DataSource`扮演了工厂模式中的抽象产品角色，`UnpooledDataSource`实现了`DataSource`接口，扮演了具体产品角色

首先看一下`UnpooledDataSource`的主要字段

```java

/** 加载driver的ClassLoader **/
private ClassLoader driverClassLoader;

/** 数据库连接的配置项 **/
private Properties driverProperties;

/** 所有注册的驱动  **/
private static Map<String, Driver> registeredDrivers = new ConcurrentHashMap<String, Driver>();

/** 数据库连接驱动名称 **/
private String driver;

/** 地址 **/
private String url;

/** 用户名 **/
private String username;

/** 密码 **/
private String password;

/** 是否自动提交 **/
private Boolean autoCommit;

/** 事务隔离级别 **/
private Integer defaultTransactionIsolationLevel;

```

在各个厂商实现的`JDBC`连接中，首先都需要向`java.sql.DriverManager`中注册自己的驱动，如`com. mysql.jdbc.Driver`有如下代码

```java
static {
    try {
        DriverManager.registerDriver(new Driver());
    } catch (SQLException var1) {
        throw new RuntimeException("Can't register driver!");
    }
}
```

在`UnpooledDataSource`的静态代码块中，首先取出`DriverManager`中注册的所有驱动，放入`registeredDrivers`中

```java
static {
    Enumeration<Driver> drivers = DriverManager.getDrivers();
    while (drivers.hasMoreElements()) {
        Driver driver = drivers.nextElement();
        registeredDrivers.put(driver.getClass().getName(), driver);
    }
}
```

`UnpooledDataSource`实现了`javax.sql.DataSource#getConnection()`方法，而这个方法最终会调用`doGetConnection()`

```java
private Connection doGetConnection(String username, String password) throws SQLException {
    Properties props = new Properties();

    //保存数据库连接配置
    if (driverProperties != null) {
        props.putAll(driverProperties);
    }

    if (username != null) {
        props.setProperty("user", username);
    }
    if (password != null) {
        props.setProperty("password", password);
    }
    return doGetConnection(props);
}

private Connection doGetConnection(Properties properties) throws SQLException {
    //如果还没有注册过此驱动，则先注册驱动
    initializeDriver();
    //获取连接
    Connection connection = DriverManager.getConnection(url, properties);
    //事务相关配置
    configureConnection(connection);
    return connection;
}

private synchronized void initializeDriver() throws SQLException {
    //如果没有注册过则开始注册
    if (!registeredDrivers.containsKey(driver)) {
        Class<?> driverType;
        try {
            if (driverClassLoader != null) {
                driverType = Class.forName(driver, true, driverClassLoader);
            } else {
                driverType = Resources.classForName(driver);
            }
            Driver driverInstance = (Driver)driverType.newInstance();
            //向java.sql.DriverManager中注册驱动
            //随后添加到registeredDrivers集合
            DriverManager.registerDriver(new DriverProxy(driverInstance));
            registeredDrivers.put(driver, driverInstance);
        } catch (Exception e) {
            throw new SQLException("....");
        }
    }
}

private void configureConnection(Connection conn) throws SQLException {

    //配置是否自动提交
    if (autoCommit != null && autoCommit != conn.getAutoCommit()) {
        conn.setAutoCommit(autoCommit);
    }

    //配置事务隔离界别
    if (defaultTransactionIsolationLevel != null) {
        conn.setTransactionIsolation(defaultTransactionIsolationLevel);
    }
}
```

### 池化连接

#### PooledDataSourceFactory

数据库连接时一个很耗费资源的操作，尤其是不断的打开关闭连接，所以在大部分的ORM框架中，都用到了数据库连接池。`MyBatis`也不例外，提供了`PooledDataSourceFactory`来创建`PooledDataSource`实现数据库连接池，`PooledDataSourceFactory`继承了`UnpooledDataSourceFactory`，只在初始化时改变了`dataSource`属性。

```java
public PooledDataSourceFactory() {
    this.dataSource = new PooledDataSource();
}
```

#### PooledDataSource

`PooledDataSource`实现了简易版连接池的功能，整体结构如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/pooleddatasourse%E7%B1%BB%E5%9B%BE.png)

`PooledDataSource`连接数据库的功能还是依赖于`UnpooledDataSource`。`PooledDataSource`作为连接池，并不会直接管理`java.sql.Connection`，而是管理`org.apache.ibatis.datasource.pooled.PooledConnection`连接对象。

#### PooledConnection

`PooledConnection`对象其内部封装了`java.sql.Connection`，并通过动态代理提供了代理对象。

`PooledConnection`主要字段有

```java

/** 要改变的方法 **/
private static final String CLOSE = "close";
private static final Class<?>[] IFACES = new Class<?>[] { Connection.class };

private int hashCode = 0;

/** 记录了当前对象所属的PooledDataSource对象 **/
private PooledDataSource dataSource;

/** Connection的真实对象 **/
private Connection realConnection;

/** Connection的代理对象 **/
private Connection proxyConnection;

/** 该连接从数据库中取出的时间 ***/
private long checkoutTimestamp;

/** 该连接创建的时间 **/
private long createdTimestamp;

/** 最后一次被使用的时间 **/
private long lastUsedTimestamp;

/** 根据数据库连接信息计算出来的hash值，用来标识连接池 **/
private int connectionTypeCode;

/** 是否可用 **/
private boolean valid;
```

该类的核心思想为：代理`java.sql.Connection`对象，当用户调用`connection.close()`方法时，不关闭连接，而是将其放入到连接池中。

既然使用了动态代理，那么核心方法肯定就是`invoke()`

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //方法名称
    String methodName = method.getName();
    //如果是connection.close()方法
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
        //放入到数据库连接池中
        dataSource.pushConnection(this);
        return null;
    } else {
        //忽略try-catch
        if (!Object.class.equals(method.getDeclaringClass())) {
          //如果调用的方法不是Object类的方法，就证明是数据库连接相关的方法，检测连接是否可用
            checkConnection();
        }
        return method.invoke(realConnection, args);
    }
}

private void checkConnection() throws SQLException {
    if (!valid) {
        throw new SQLException("....");
    }
}
```



#### PoolState

`PoolState`是用来管理`PooledConnection`连接状态的组件，此对象中也存放了数据库连接，首先看一下主要的字段

```java
/**
   * 此对象对应的PooledDataSource对象
   * **/
protected PooledDataSource dataSource;

/** 空闲连接  **/
protected final List<PooledConnection> idleConnections = new ArrayList<PooledConnection>();

/** 活跃连接  **/
protected final List<PooledConnection> activeConnections = new ArrayList<PooledConnection>();

/** 请求数据库连接的次数  **/
protected long requestCount = 0;

/** 获取连接的累计时间  **/
protected long accumulatedRequestTime = 0;

  /** CheckoutTime表示从连接池中取出连接，到归还连接的这段时长
   *  accumulatedCheckoutTime为CheckoutTime的总时长
   * **/
protected long accumulatedCheckoutTime = 0;

  /** 当连接长时间未归还连接池时，会被认为是超时
   *   claimedOverdueConnectionCount代表超时的连接个数
   * **/
protected long claimedOverdueConnectionCount = 0;

/** 累积超时时间  **/
protected long accumulatedCheckoutTimeOfOverdueConnections = 0;

/** 累积等待时间  **/
protected long accumulatedWaitTime = 0;

/** 等待次数  **/
protected long hadToWaitCount = 0;

/** 无效的连接数  **/
protected long badConnectionCount = 0;
```

#### PooledDataSource

前面说过，`PooledDataSource`连接数据库的功能还是依赖于其内部封装的`UnpooledDataSource`，并且由`PoolState`管理所有的状态。`PooledDataSource`的主要字段有

```java
private final PoolState state = new PoolState(this);

private final UnpooledDataSource dataSource;

/** 最大活跃连接数量 **/
protected int poolMaximumActiveConnections = 10;

/** 最大空闲连接数量 **/
protected int poolMaximumIdleConnections = 5;

/** 最大checkout时长 **/
protected int poolMaximumCheckoutTime = 20000;

/** 无法获得连接时，线程需要等待的时间 **/
protected int poolTimeToWait = 20000;

/** 测试连接是否有效时使用的SQL **/
protected String poolPingQuery = "NO PING QUERY SET";

/** 是否发送测试SQL **/
protected boolean poolPingEnabled;

/** 连接超过poolPingConnectionsNotUsedFor毫秒未使用时，会发送一次测试SQL语句来检测连接是否可用 **/
protected int poolPingConnectionsNotUsedFor;

  /**
   * 根据数据库的URL 、用户名和密码生成的一个hash值，该哈希值用于标志着当前的连接池，在构造函数中初始化
   * **/
private int expectedConnectionTypeCode;
```



`PooledDataSource.getConnection()`获取连接对象时，会调用`popConnection()`方法获取`PooledConnection`对象，然后再获取其内部封装的代理对象。

```java
public Connection getConnection() throws SQLException {
    return popConnection(dataSource.getUsername(), dataSource.getPassword()).getProxyConnection();
}
```

`popConnection()`方法是`PooledDataSource`类的核心方法之一，用来从连接池中获取`PooledConnection`连接对象，主要思路如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/popConnection.png)

方法具体实现如下（以下代码省略`try-catch`和日志记录）

```java
private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    //循环获取连接对象
    while (conn == null) {
        synchronized (state) {
            //如果空闲连接不是空，则直接取出
            if (!state.idleConnections.isEmpty()) {
                conn = state.idleConnections.remove(0);
            } else {
                //如果是空，则检测活跃连接是否达到了最大活跃连接数
                if (state.activeConnections.size() < poolMaximumActiveConnections) {
                    // 如果没有达到则证明可以创建新连接
                    conn = new PooledConnection(dataSource.getConnection(), this);
                } else {
                    // 如果达到了最大连接数，判断是否有超时连接
                    //取出连接对象
                    PooledConnection oldestActiveConnection = state.activeConnections.get(0);
                    //取出该连接的checkout时间
                    long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
                    //如果该时间大于最大超时时间，则证明已经是超时连接
                    if (longestCheckoutTime > poolMaximumCheckoutTime) {
                        //统计总超时时间并且将其从活动连接中移除
                        state.claimedOverdueConnectionCount++;
                        state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
                        state.accumulatedCheckoutTime += longestCheckoutTime;
                        state.activeConnections.remove(oldestActiveConnection);
                        //如果当前连接不是自动提交，将还没有处理完的操作回滚
                        if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                            oldestActiveConnection.getRealConnection().rollback();
                        }
                        //将此连接重新初始化
                        conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
                        conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
                        conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
                        //将连接变为不可用
                        oldestActiveConnection.invalidate();
                    } else {
                        //如果没有超时连接，就将当前线程阻塞等待
                        if (!countedWait) {
                            state.hadToWaitCount++;
                            countedWait = true;
                        }
                        long wt = System.currentTimeMillis();
                        state.wait(poolTimeToWait);
                        state.accumulatedWaitTime += System.currentTimeMillis() - wt;
                    }
                }
            }
            if (conn != null) {
                if (conn.isValid()) {
                    //如果当前连接可用，先将之前未完成的操作回滚
                    if (!conn.getRealConnection().getAutoCommit()) {
                        conn.getRealConnection().rollback();
                    }
                    //设置hash值和时间
                    conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
                    conn.setCheckoutTimestamp(System.currentTimeMillis());
                    conn.setLastUsedTimestamp(System.currentTimeMillis());
                    //将当前连接加入到活动连接中，统计数据
                    state.activeConnections.add(conn);
                    state.requestCount++;
                    state.accumulatedRequestTime += System.currentTimeMillis() - t;
                } else {
                    //如果不可用则统计失效数据
                    state.badConnectionCount++;
                    localBadConnectionCount++;
                    conn = null;
                    if (localBadConnectionCount > (poolMaximumIdleConnections + 3)) {
                        //如果失败次数大于最大空闲数加3，则抛出异常
                        throw new SQLException("....");
                    }
                }
            }
        }

    }

    if (conn == null) {
        throw new SQLException("....");
    }

    return conn;
}
```



前面分析`PooledConnection.invoke()`方法时，我们看到，当调用`connection.close()`方法时，并不会关闭连接，而是调用`PooledDataSource.pushConnection()`方法将当前连接放回数据库连接池中。

`pushConnection()`方法也是`PooledDataSource`类的核心方法之一，方法逻辑如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/pushConnection.png)



方法具体实现如下（以下代码省略`try-catch`和日志记录）



```java
protected void pushConnection(PooledConnection conn) throws SQLException {

    synchronized (state) {
        //从活动连接中移除此连接
        state.activeConnections.remove(conn);
        //判断连接是否可用
        if (conn.isValid()) {
            //如果空闲连接小于最大空闲连接数
            //并且是此连接池创建的数据库连接
            if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                //统计checkout时间
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                //如果不是自动提交，将之前的操作回滚
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }
                //将此连接加入到空闲连接队列中
                PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
                state.idleConnections.add(newConn);
                //设置时间及不可用标识
                newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
                newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
                conn.invalidate();
                //通知阻塞的线程可以重新获取连接了
                state.notifyAll();
            } else {
                //如果进入此else，证明不能放入空闲连接集合中，要将此连接关闭
                //统计checkout时间
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                //如果不是自动提交，将之前的操作回滚
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }
                //关闭连接
                conn.getRealConnection().close();
                conn.invalidate();
            }
        } else {
            //如果连接不可用，统计失效连接数
            state.badConnectionCount++;
        }
    }
}
```



在上述的两个方法中，都调用了`conn.isValid()`判断连接是否可用，该方法除了判断`valid`字段的值外还会调用`pingConnection()`方法测试连接是否可用。

```java
public boolean isValid() {
    return valid && realConnection != null && dataSource.pingConnection(this);
}
```

`pingConnection()`主要逻辑是创建一个`Statement`来执行`SQL`，如果成功执行则证明方法可用，实现如下

```java
protected boolean pingConnection(PooledConnection conn) {
    boolean result = true;

    try {
        //判断该连接是否已经关闭
        result = !conn.getRealConnection().isClosed();
    } catch (SQLException e) {
        result = false;
    }

    //如果连接没有关闭
    if (result) {
        //如果开启了发送测试SQL功能
        if (poolPingEnabled) {
            //如果已经达到了该发送测试SQL的时间
            if (poolPingConnectionsNotUsedFor >= 0 && conn.getTimeElapsedSinceLastUse() > poolPingConnectionsNotUsedFor) {
                try {
                    //创建Statement执行SQL，如果没有异常则证明连接可用
                    Connection realConn = conn.getRealConnection();
                    Statement statement = realConn.createStatement();
                    ResultSet rs = statement.executeQuery(poolPingQuery);
                    rs.close();
                    statement.close();
                    if (!realConn.getAutoCommit()) {
                        realConn.rollback();
                    }
                    result = true;
                } catch (Exception e) {
                    try {
                        conn.getRealConnection().close();
                    } catch (Exception e2) {
                       
                    }
                    result = false;
                }
            }
        }
    }
    return result;
}
```



最后需要注意的是`PooledDataSource.forceCloseAll()`方法，在更改了数据库配置（如用户名、密码、超时时间等）后，会调用此方法关闭所有连接，并清空可用连接和空闲连接集合。随后获取连接时，会重新初始化

```java
public void forceCloseAll() {
    synchronized (state) {
        //生成新的hash值
        expectedConnectionTypeCode = assembleConnectionTypeCode(dataSource.getUrl(), dataSource.getUsername(), dataSource.getPassword());
        //遍历两个集合
        for (int i = state.activeConnections.size(); i > 0; i--) {
            try {
                //移除此连接
                PooledConnection conn = state.activeConnections.remove(i - 1);
                //置为不可用
                conn.invalidate();

                Connection realConn = conn.getRealConnection();
                //将事务回滚后关闭
                if (!realConn.getAutoCommit()) {
                    realConn.rollback();
                }
                realConn.close();
            } catch (Exception e) {
            }
        }
        //同样逻辑处理state.idleConnections集合
    }
}
```

















