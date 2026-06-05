---
title: MyBatis源码分析(14)-基础支持层之事务管理
tags:
  - MyBatis
date: 2018-12-16
categories:
 - MyBatis
---

## 事务管理

在实践开发中， 控制数据库事务是一件非常重要的工作， `MyBatis`使用`Transaction` 接口对
数据库事务进行了抽象，`MyBatis`的事务模块使用了工厂模式，首先来看一下整体结构

![](/images/oss/%E5%8D%9A%E5%AE%A2/%E4%BA%8B%E5%8A%A1%E5%B7%A5%E5%8E%82%E6%A8%A1%E5%BC%8F.png)

其中，`Transaction`就是工厂模式中的产品接口，`TransactionFactory`是抽象工厂角色。

`ManagedTransactionFactory`和`JdbcTransactionFactory`是具体工厂角色。

`JdbcTransaction`和`ManagedTransaction`是具体产品角色

### Transaction

 `Transaction`接口的定义如下：

```java
public interface Transaction {

    /**
   * 获取数据库连接
   */
    Connection getConnection() throws SQLException;

    /**
   * 提交事务
   */
    void commit() throws SQLException;

    /**
   * 回滚事务
   */
    void rollback() throws SQLException;

    /**
   * 关闭数据库连接
   */
    void close() throws SQLException;

    /**
   * 获取事务超时时间
   */
    Integer getTimeout() throws SQLException;

}
```

前面介绍过，`Transaction`接口有两个实现类，`JdbcTransaction`和`ManagedTransaction`。

### JdbcTransaction

`JdbcTransaction`直接使用`JDBC`提交和回滚功能。它依赖于从`dataSource`获取的连接来管理事务。 延迟加载`connection`，直到调用`getConnection()`。启用自动提交时忽略提交或回滚请求。

`JdbcTransaction`的主要字段如下

```java
/** 此事务所属的连接 **/
protected Connection connection;

/** 连接所属的数据源 **/
protected DataSource dataSource;

/** 数据隔离级别 **/
protected TransactionIsolationLevel level;

/** 是否开启自动提交 **/
protected boolean autoCommmit;
```

`JdbcTransaction`构造方法中初始化了除`connection`之外的字段，而对`connection`采取了延迟加载策略。

```java
public JdbcTransaction(DataSource ds, TransactionIsolationLevel desiredLevel, boolean desiredAutoCommit) {
    dataSource = ds;
    level = desiredLevel;
    autoCommmit = desiredAutoCommit;
}

@Override
public Connection getConnection() throws SQLException {
    if (connection == null) {
        openConnection();
    }
    return connection;
}

protected void openConnection() throws SQLException {
    connection = dataSource.getConnection();
    if (level != null) {
        //设置事务隔离级别
        connection.setTransactionIsolation(level.getLevel());
    }
    //设置自动提交
    setDesiredAutoCommit(autoCommmit);
}
```

`JdbcTransaction`的`commit()、rollback()`等方法都是通过调用`JDBC`的API实现的，逻辑比较简单，这里只看一个方法，感兴趣的同学可以自行查看源码

```java
public void commit() throws SQLException {
    if (connection != null && !connection.getAutoCommit()) {
        connection.commit();
    }
}
```



`ManagedTransaction`的实现更为简单，这里就不再分析了。



`TransactionFactory`

`TransactionFactory`类是抽象工厂接口，负责创建`Transaction`对象，该类的方法主要有

```java
public interface TransactionFactory {

    /**
   * 配置TransactionFactory属性
   */
    void setProperties(Properties props);

    /**
   * 创建新连接
   */
    Transaction newTransaction(Connection conn);

    /**
   * 从指定的DataSource中创建新连接，指定事务隔离级别和是否自动提交
   */
    Transaction newTransaction(DataSource dataSource, TransactionIsolationLevel level, boolean autoCommit);

}
```

`ManagedTransactionFactory`和`JdbcTransactionFactory`两个实现类比较简单，这里不再描述。

在实践中，` MyBatis `通常会与`Spring` 集成使用， 数据库的事务是交给`Spring` 进行管理的。

















