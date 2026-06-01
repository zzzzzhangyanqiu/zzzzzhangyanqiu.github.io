---
title: MyBatis源码分析(27)-核心处理层之StatementHandler
tags:
  - MyBatis
date: 2019-04-28
categories:
 - MyBatis
---

### StatementHandler

`StatementHandler`是`MyBatis`中比较核心的接口之一。此模块完成的功能都是比较核心的功能，如数据库的查询，修改，增加等等。同时该接口也是后面要分析的`Executor`的实现基础。

首先来看此接口的`UML`类图

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/MyBatis/StatementHandler%E7%B1%BB%E5%9B%BE.png)

可以看到`StatementHandler`接口由一个抽象类和`RoutingStatementHandler`实现，这里`MyBatis`用到了策略模式，如果你对此模式还不是很熟悉，建议看看[这篇文章](https://suiyueranzly.gitee.io/posts/3932426784/)或者去网上找一些相关的资料，下面是该接口定义的方法

```java
/**
   * 创建Statement
   * **/
Statement prepare(Connection connection, Integer transactionTimeout)
    throws SQLException;

/**
   * 绑定实参
   * **/
void parameterize(Statement statement)
    throws SQLException;

/**
   * 执行批量操作
   * **/
void batch(Statement statement)
    throws SQLException;

/**
   * 创建Statement
   * **/
int update(Statement statement)
    throws SQLException;

/**
   * 查询
   * **/
<E> List<E> query(Statement statement, ResultHandler resultHandler)
    throws SQLException;

/**
   * 查询游标对象
   * **/
<E> Cursor<E> queryCursor(Statement statement)
    throws SQLException;

/**
   * 获取BoundSql对象
   * **/
BoundSql getBoundSql();

/**
   * 获取ParameterHandler对象，该对象的作用稍后会进行分析
   * **/
ParameterHandler getParameterHandler();
```

首先来介绍`RoutingStatementHandler`

#### RoutingStatementHandler

在上图中我们可以看到`StatementHandler`整个模块的结构图，其中`RoutingStatementHandler`就扮演了策略模式中的环境角色，该类的字段只有一个，就是`private final StatementHandler delegate;`，并且在该类的构造方法中，会根据`statementType`的不同分别创建实现类，如下

```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

    switch (ms.getStatementType()) {
        case STATEMENT:
            //如果是普通类型
            delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
            break;
        case PREPARED:
            //如果是prepared类型
            delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
            break;
        case CALLABLE:
            //如果是callable类型
            delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
            break;
        default:
            throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

}
```

`RoutingStatementHandler`的其它方法都是调用`delegate`的对应方法，这里就不在分析了，接下来我们分析`StatementHandler`接口的另外一个实现类`BaseStatementHandler`

#### BaseStatementHandler

`BaseStatementHandler`类是一个抽象类，提供了一些公共方法供三个子类进行调用，其字段如下

```java
/***
   * 全局Configuration对象
   * **/
protected final Configuration configuration;
/***
   * 全局ObjectFactory对象，用来创建类
   * **/
protected final ObjectFactory objectFactory;
/***
   * TypeHandlerRegistry对象 用来获取TypeHandler
   * **/
protected final TypeHandlerRegistry typeHandlerRegistry;
/***
   * 用来处理结果集
   * **/
protected final ResultSetHandler resultSetHandler;
/***
   * 用来处理参数
   * **/
protected final ParameterHandler parameterHandler;

/**
   * 执行器，后面会进行介绍
   * **/
protected final Executor executor;
protected final MappedStatement mappedStatement;
/***
   * RowBounds对象，其中封装了offset，limit信息
   * **/
protected final RowBounds rowBounds;

/**
   * BoundSql对象，封装了SQL语句相关的信息
   * **/
protected BoundSql boundSql;
```

该类的构造方法中，会生成主键并创建`parameterHandler/resultSetHandler`对象

```java
protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    //省略。。。

    if (boundSql == null) {
        //生成主键
        generateKeys(parameterObject);
        boundSql = mappedStatement.getBoundSql(parameterObject);
    }

    this.boundSql = boundSql;

    //创建parameterHandler和resultSetHandler对象
    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
}
```

`generateKeys()`会调用我们前面分析过的`KeyGenerator`来生成主键

```java
protected void generateKeys(Object parameter) {
    //获取KeyGenerator
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    ErrorContext.instance().store();
    //调用方法生成主键
    keyGenerator.processBefore(executor, mappedStatement, null, parameter);
    ErrorContext.instance().recall();
}
```

`StatementHandler`依赖于`ParameterHandler`处理参数，首先来分析一下`ParameterHandler`

##### ParameterHandler

`ParameterHandler`的大概原理就是调用`Statement.set*()`方法将用户传入的实参替换为`SQL`中的`?`，`ParameterHandler`是一个接口，其方法如下

```java
/**
   * 获取用户传入的实参
   * **/
Object getParameterObject();

/**
   * 对Statement中的参数赋值
   * **/
void setParameters(PreparedStatement ps)
    throws SQLException;
```

`MyBatis`为该接口提供了一个默认的实现类`DefaultParameterHandler`，首先来看`DefaultParameterHandler`的字段

```java
/**
   * 用来获取TypeHandler
   * **/
private final TypeHandlerRegistry typeHandlerRegistry;

/**
   * 对应的MappedStatement对象
   * ***/
private final MappedStatement mappedStatement;
/***
   * 用户传入的实参
   * **/
private final Object parameterObject;
/***
   * 存储了SQL的信息
   * **/
private BoundSql boundSql;
private Configuration configuration;
```

`DefaultParameterHandler.setParameters()`方法会根据参数类型选择不同的`TypeHandler`进行处理

```java
public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    //获取SQL中的参数集合
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
        //遍历
        for (int i = 0; i < parameterMappings.size(); i++) {
            ParameterMapping parameterMapping = parameterMappings.get(i);
            //过滤掉输出类型的参数
            if (parameterMapping.getMode() != ParameterMode.OUT) {
                Object value;
                //获取此参数的名称
                String propertyName = parameterMapping.getProperty();
                //校验用户是否传入了此参数
                if (boundSql.hasAdditionalParameter(propertyName)) {
                    //获取实参
                    value = boundSql.getAdditionalParameter(propertyName);
                } else if (parameterObject == null) {
                    //如果用户参入的实参为空，则值为空
                    value = null;
                } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                    //如果有此实参类型的TypeHandler，证明是基础类型，则直接使用此值
                    value = parameterObject;
                } else {
                    //到了此判断分支，证明用户传入的是bean类型或者是map集合，使用MetaObject获取值
                    MetaObject metaObject = configuration.newMetaObject(parameterObject);
                    value = metaObject.getValue(propertyName);
                }
                //调用对应的TypeHandler进行SQL语句的参数赋值
                TypeHandler typeHandler = parameterMapping.getTypeHandler();
                JdbcType jdbcType = parameterMapping.getJdbcType();
                if (value == null && jdbcType == null) {
                    jdbcType = configuration.getJdbcTypeForNull();
                }
                try {
                    typeHandler.setParameter(ps, i + 1, value, jdbcType);
                } catch (TypeException e) {
                    throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
                } catch (SQLException e) {
                    throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
                }
            }
        }
    }
}
```

回到`BaseStatementHandler`的分析。前面说过，`BaseStatementHandler`为其子类提供了一些公共的方法，其`prepare()`创建了`Statement`并设置超时时间等，实现如下

```java
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
        //初始化Statement对象，instantiateStatement方法是一个抽象方法，交给子类去实现
        statement = instantiateStatement(connection);
        //设置超时时间
        setStatementTimeout(statement, transactionTimeout);
        //设置statement的fetchSize
        setFetchSize(statement);
        return statement;
    } catch (SQLException e) {
        closeStatement(statement);
        throw e;
    } catch (Exception e) {
        closeStatement(statement);
        throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
}
```

其中`setStatementTimeout()/setFetchSize`实现比较简单，这里就不在分析了，下面开始分析三个子类。

##### SimpleStatementHandler

`SimpleStatementHandler`负责使用`Statement`对象执行操作。`instantiateStatement()`就是创建一个普通的`Statement`对象

```java
protected Statement instantiateStatement(Connection connection) throws SQLException {
    //根据resultSetType创建Statement
    if (mappedStatement.getResultSetType() != null) {
        return connection.createStatement(mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
        return connection.createStatement();
    }
}
```

由于`Statement`不能执行带有`?`的`SQL`语句，所以此类执行的一定都是可以直接运行的`SQL`，因此，此类的`parameterize()`方法是空实现。

`SimpleStatementHandler.batch()/query()/queryCursor()`三个方法都是调用`Statement`的`API`进行操作，以`query()`举例

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    //获取sql语句
    String sql = boundSql.getSql();
    //执行
    statement.execute(sql);
    //转换结果集
    return resultSetHandler.<E>handleResultSets(statement);
}
```

其余两个方法类似，就不再进行分析了

`SimpleStatementHandler.update()`会执行数据库的增加、修改等操作

```java
public int update(Statement statement) throws SQLException {
    //SQL语句
    String sql = boundSql.getSql();
    //获取用户传入的实参
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    int rows;
    //调用KeyGenerator.processAfter()方法处理主键
    if (keyGenerator instanceof Jdbc3KeyGenerator) {
        statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
        rows = statement.getUpdateCount();
        keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else if (keyGenerator instanceof SelectKeyGenerator) {
        statement.execute(sql);
        rows = statement.getUpdateCount();
        keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else {
        statement.execute(sql);
        rows = statement.getUpdateCount();
    }
    //返回受影响的行数
    return rows;
}
```

##### PreparedStatementHandler

根据名字可以看出，`PreparedStatementHandler`是使用`PreparedStatement`进行操作的，所以其`instantiateStatement`就是创建一个`PreparedStatement`对象

```java
protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
        //如果是Jdbc3KeyGenerator，则创建RETURN_GENERATED_KEYS属性的prepareStatement
        String[] keyColumnNames = mappedStatement.getKeyColumns();
        if (keyColumnNames == null) {
            return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
        } else {
            return connection.prepareStatement(sql, keyColumnNames);
        }
    } else if (mappedStatement.getResultSetType() != null) {
        //根据ResultSetType创建
        return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
        //创建
        return connection.prepareStatement(sql);
    }
}
```

其`parameterize()`方法会调用前面分析过的`parameterHandler`处理参数

```java
public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
}
```

其余方法与``SimpleStatementHandler()`实现类似，这里就不再进行分析了

##### CallableStatementHandler

`CallableStatementHandler`使用`CallableStatement`进行操作，其`instantiateStatement()`实现如下

```java
protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    //根据ResultSetType创建CallableStatement
    if (mappedStatement.getResultSetType() != null) {
        return connection.prepareCall(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    } else {
        return connection.prepareCall(sql);
    }
}
```

同样，`parameterize()`方法也会调用`parameterHandler`处理参数，不同的是该方法还会处理输出类型的参数

```java
public void parameterize(Statement statement) throws SQLException {
    //处理输出参数，原理就是调用CallableStatement.registerOutParameter()方法
    registerOutputParameters((CallableStatement) statement);
    parameterHandler.setParameters((CallableStatement) statement);
}
```

其余方法与其它两个类实现原理类似，这里就不再进行分析了。

