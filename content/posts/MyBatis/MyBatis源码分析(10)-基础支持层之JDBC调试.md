---
title: MyBatis源码分析(10)-基础支持层之JDBC调试
tags:
  - MyBatis
date: 2018-10-20
categories:
 - MyBatis
---

## JDBC调试

再了解JDBC调试之前，我们先来看一段`MyBatis`的输出日志

```sql
==>  Preparing: select * from user_info user_id = ? 
==> Parameters: 123456(Integer)
<==      Total: 1
```

如果日常中你使用的是`MyBatis`，那上面的日志你应该每天都会看到，`JDBC`调试主要的作用就是在开发阶段，将`JDBC`操作通过对应的日志框架打印出来。例如上面的日志信息，其中就包含了用户执行的`SQL`、传入的参数和返回结果行数（如果是`update`等操作则返回受影响的行数）。`JDBC`调试功能主要通过`JDK`提供的动态代理实现，所以如果你还不是很清楚动态代理模式，可以先看一下[这篇文章](https://suiyueranzly.gitee.io/posts/3822496321/)。

### BaseJdbcLogger

`BaseJdbcLogger`是一个抽象类，它是`JDBC`包下所有`logger`类的父类，类关系图如下

![](/images/oss/%E5%8D%9A%E5%AE%A2/BaseJdbcLogger%E7%B1%BB%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

看到`InvocationHandler`接口，可能有的同学立刻就会想到`JDK`的动态代理，别着急，我们一步步的来分析。

首先来介绍`BaseJdbcLogger`类几个核心属性

```java
//记录了preparedStatement的set方法集合
protected static final Set<String> SET_METHODS = new HashSet<String>();
//记录了preparedStatement的execute方法集合
protected static final Set<String> EXECUTE_METHODS = new HashSet<String>();


//preparedStatement的set方法设置的键值对
private Map<Object, Object> columnMap = new HashMap<Object, Object>();

//preparedStatement的set方法设置的key值
private List<Object> columnNames = new ArrayList<Object>();

//preparedStatement的set方法设置的value值
private List<Object> columnValues = new ArrayList<Object>();

//用于输出日志的Log对象
//还记得我们之前的文章么，各个日志框架就是通过此接口进行适配的
protected Log statementLog;

//记录了SQL的层数，用于格式化sql
protected int queryStack;
```

`BaseJdbcLogger`类的`static`块中为`SET_METHODS`和`EXECUTE_METHODS`中添加了常用的`preparedStatement.set()`方法和`preparedStatement.execute()`方法

```java
static {
    SET_METHODS.add("setString");
    SET_METHODS.add("setNString");
    SET_METHODS.add("setInt");
    //省略其它set*方法

    EXECUTE_METHODS.add("execute");
    EXECUTE_METHODS.add("executeUpdate");
    EXECUTE_METHODS.add("executeQuery");
    EXECUTE_METHODS.add("addBatch");
}
```

`BaseJdbcLogger`类中提供了填充上述集合的方法和一些工具方法，这些稍后会讲解。

### ConnectionLogger

`ConnectionLogger`是`BaseJdbcLogger`的一个子类，实现了`InvocationHandler`接口，此类中封装了`java.sql.Connection`对象。`ConnectionLogger.newInstance()`方法使用了`JDK`的动态代理返回`java.sql.Connection`对象的代理类。

```java
public static Connection newInstance(Connection conn, Log statementLog, int queryStack) {
    InvocationHandler handler = new ConnectionLogger(conn, statementLog, queryStack);
    ClassLoader cl = Connection.class.getClassLoader();
    return (Connection) Proxy.newProxyInstance(cl, new Class[]{Connection.class}, handler);
}
```

上述代码是标准的动态代理使用方式，相信了解动态代理模式的同学看起来都不会有问题。由于实现了`InvocationHandler`接口，所以`invoke()`方法是`ConnectionLogger`类的核心逻辑，实现如下

```java
@Override
public Object invoke(Object proxy, Method method, Object[] params)
    throws Throwable {
    try {
        //如果调用是object的方法，如equals()、toString()等，则直接执行
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, params);
        }  
        //如果是prepareStatement方法，则创建PreparedStatement的代理类并输出日志
        if ("prepareStatement".equals(method.getName())) {
            if (isDebugEnabled()) {
                //例如，我们调用PreparedStatement preparedStatement = conn.prepareStatement("select * from user_info");
                //则会输出Preparing: select * from user_info也就是我们看到的日志了
                //removeBreakingWhitespace()主要作用就是去掉参数中的多余空格、空行等等。
                debug(" Preparing: " + removeBreakingWhitespace((String) params[0]), true);
            }
            PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);
            stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);
            return stmt;
        } else if ("prepareCall".equals(method.getName())) {
            if (isDebugEnabled()) {
                debug(" Preparing: " + removeBreakingWhitespace((String) params[0]), true);
            }
            PreparedStatement stmt = (PreparedStatement) method.invoke(connection, params);
            stmt = PreparedStatementLogger.newInstance(stmt, statementLog, queryStack);
            return stmt;
        } else if ("createStatement".equals(method.getName())) {
            Statement stmt = (Statement) method.invoke(connection, params);
            stmt = StatementLogger.newInstance(stmt, statementLog, queryStack);
            return stmt;
        } else {
            return method.invoke(connection, params);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}
```

### PreparedStatementLogger

`PreparedStatementLogger`类中封装了`PreparedStatement`对象，并实现了`InvocationHandler`接口。同理`PreparedStatementLogger.newInstance()`方法同样返回了`PreparedStatement`对象的动态代理类，这里就不在过多阐述，我们还是把重心放在`PreparedStatementLogger.invoke()`核心方法上面。

```java
@Override
public Object invoke(Object proxy, Method method, Object[] params) throws Throwable {
    try {
        //如果调用是object的方法，如equals()、toString()等，则直接执行
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, params);
        }
        //如果调用的是执行方法，则输出Parameters: 日志
        if (EXECUTE_METHODS.contains(method.getName())) {
            if (isDebugEnabled()) {
                debug("Parameters: " + getParameterValueString(), true);
            }
            //因为每执行一次sql，参数都会不同，清除columnMap、columnNames、columnValues三个集合
            clearColumnInfo();
            //如果调用的是executeQuery()方法，则证明是一个查询方法，查询方法要返回ResultSet
            //这里查询完毕后直接调用ResultSetLogger.newInstance()返回ResultSet的代理对象
            if ("executeQuery".equals(method.getName())) {
                ResultSet rs = (ResultSet) method.invoke(statement, params);
                return rs == null ? null : ResultSetLogger.newInstance(rs, statementLog, queryStack);
            } else {
                return method.invoke(statement, params);
            }
            //如果调用的是set*()方法，则将值保存在columnMap、columnNames、columnValues三个集合中
        } else if (SET_METHODS.contains(method.getName())) {
            if ("setNull".equals(method.getName())) {
                setColumn(params[0], null);
            } else {
                setColumn(params[0], params[1]);
            }
            return method.invoke(statement, params);
            //如果执行的是getResultSet()方法，
            //则调用ResultSetLogger.newInstance()返回ResultSet的代理对象
        } else if ("getResultSet".equals(method.getName())) {
            ResultSet rs = (ResultSet) method.invoke(statement, params);
            return rs == null ? null : ResultSetLogger.newInstance(rs, statementLog, queryStack);
            //如果是执行的是getUpdateCount()方法来计算影响行数，则除了返回结果还会记录 Updates: 日志
        } else if ("getUpdateCount".equals(method.getName())) {
            int updateCount = (Integer) method.invoke(statement, params);
            if (updateCount != -1) {
                debug("   Updates: " + updateCount, false);
            }
            return updateCount;
        } else {
            return method.invoke(statement, params);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}

//此方法会遍历columnValues集合，获取所有的参数值并返回
protected String getParameterValueString() {
    List<Object> typeList = new ArrayList<Object>(columnValues.size());
    for (Object value : columnValues) {
        if (value == null) {
            typeList.add("null");
        } else {
            typeList.add(objectValueString(value) + "(" + value.getClass().getSimpleName() + ")");
        }
    }
    final String parameters = typeList.toString();
    return parameters.substring(1, parameters.length() - 1);
}
```

### ResultSetLogger

`ResultSetLogger`中封装了`ResultSet`对象，并实现了`InvocationHandler`接口。首先看一下该类的核心字段。

```java
//记录了超大长度的类型
private static Set<Integer> BLOB_TYPES = new HashSet<Integer>();

//是否是ResultSet的第一行
private boolean first = true;

//统计行数
private int rows;

//封装的ResultSet对象
private ResultSet rs;

//记录了超大字段的列编号
private Set<Integer> blobColumns = new HashSet<Integer>();
```

`ResultSetLogger`的`static`块中向`BLOB_TYPES`集合中添加了超大的类型

```java
static {
    BLOB_TYPES.add(Types.BINARY);
    BLOB_TYPES.add(Types.BLOB);
    //省略CLOB、LONGNVARCHAR、LONGVARBINARY等类型
}
```

`ResultSetLogger.newInstance()`方法与上面实现类似，这里不在阐述，继续来看一下核心的`invoke()`方法

```java
@Override
public Object invoke(Object proxy, Method method, Object[] params) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, params);
        }    
        Object o = method.invoke(rs, params);
        //如果调用是next()方法
        if ("next".equals(method.getName())) {
            //如果还有下一条数据
            if (((Boolean) o)) {
                //行数++
                rows++;
                if (isTraceEnabled()) {
                    ResultSetMetaData rsmd = rs.getMetaData();
                    //获取数据集的列数
                    final int columnCount = rsmd.getColumnCount();
                    //如果是第一行数据则输出表头
                    if (first) {
                        first = false;
                        //获取表头数据并填充blobColumns集合
                        printColumnHeaders(rsmd, columnCount);
                    }
                    //输出该行数据，如果是数据比较大的列类型，则不会将内容记录到日志中，而是记录一个"<<BLOB>>"
                    printColumnValues(columnCount);
                }
            } else {
                //如果没有下一行数据则直接输出结果
                debug("     Total: " + rows, false);
                System.out.println("     Total: " + rows);
            }
        }
        //清除有关参数的三个集合数据
        clearColumnInfo();
        return o;
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}
```

### 总结

看完了源码后，有的同学心里可能就有一些思路了，这里总结回顾一下。

`BaseJdbcLogger`是一个抽象类，提供了`preparedStatement`的参数集合，`preparedStatement.set()`方法集合和`preparedStatement.exxecute()`方法集合等，并提供了日志对象来方面子类记录日志。

`ConnectionLogger`代理了`connection`对象，主要用来创建`statement`、`prepareStatement`等的代理对象并输出执行的`SQL`日志。

`PreparedStatementLogger`代理了`PreparedStatement`对象，用于记录用户输入的参数日志和创建`ResultSet`的代理对象，

`ResultSetLogger`代理了`ResultSet`对象，用于记录返回结果行数和结果数据。













