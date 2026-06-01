---
title: MyBatis源码分析(26)-核心处理层之KeyGenerator
tags:
  - MyBatis
date: 2019-04-22
categories:
 - MyBatis
---

### KeyGenerator

在`MyBatis`中，`insert`方法主要返回受影响的行数，但是很多时候，我们都需要返回插入的主键，这时候就需要借助`MyBatis`的`KeyGenerator`功能。对于支持生成主键的数据库（比如` MySQL` 和 `SQL Server`）,`MyBatis`可以自动获取到主键并赋值，对于不支持生成主键的数据库（如`Oracle`），也可以使用`selectKey `方式获取主键。

`KeyGenerator`是一个接口，有两个方法，分别是

```java
/**
   * 在SQL语句之前执行
   * **/
void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter);

/**
   * 在SQL语句之后执行
   * **/
void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter);
```



`MyBatis`为其提供了三个实现类，`UML`图如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/MyBatis/KeyGenerator%E7%B1%BB%E5%9B%BE.png)

由于`NoKeyGenerator`方法是空实现，故不分析，主要来看`Jdbc3KeyGenerator`和`SelectKeyGenerator`

#### Jdbc3KeyGenerator

`Jdbc3KeyGenerator`主要负责处理可自动生成主键的数据库，该类的`processBefore()`方法为空实现，这里主要关注`processAfter()`方法，该方法实现如下

```java
public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    processBatch(ms, stmt, getParameters(parameter));
}
```

其中，`getParameters()`方法会将用户传入的参数转换为`Collection<Object>`类型返回

```java
private Collection<Object> getParameters(Object parameter) {
    Collection<Object> parameters = null;
    //如果是Collection类型，则直接转换
    if (parameter instanceof Collection) {
        parameters = (Collection) parameter;
    } else if (parameter instanceof Map) {
        //如果是map类型，则获取key为collection的元素
        Map parameterMap = (Map) parameter;
        if (parameterMap.containsKey("collection")) {
            parameters = (Collection) parameterMap.get("collection");
        } else if (parameterMap.containsKey("list")) {
            //如果是List类型，直接进行转换
            parameters = (List) parameterMap.get("list");
        } else if (parameterMap.containsKey("array")) {
            //如果是数组类型，则转换为数组
            parameters = Arrays.asList((Object[]) parameterMap.get("array"));
        }
    }
    if (parameters == null) {
        //如果前面都没有判断成功，则直接将参数添加进parameters中
        parameters = new ArrayList<Object>();
        parameters.add(parameter);
    }
    return parameters;
}
```

`processBatch()`方法负责获取生成的主键，并赋值到用户传入的参数中

```java
public void processBatch(MappedStatement ms, Statement stmt, Collection<Object> parameters) {
    ResultSet rs = null;
    try {
        //获取生成的主键
        rs = stmt.getGeneratedKeys();
        final Configuration configuration = ms.getConfiguration();
        final TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
        //获取用户配置的keyProperty
        final String[] keyProperties = ms.getKeyProperties();
        final ResultSetMetaData rsmd = rs.getMetaData();
        TypeHandler<?>[] typeHandlers = null;
        if (keyProperties != null && rsmd.getColumnCount() >= keyProperties.length) {
            //遍历传入的参数
            for (Object parameter : parameters) {
                if (!rs.next()) {
                    break;
                }
                //获取此参数的MetaObject对象
                final MetaObject metaParam = configuration.newMetaObject(parameter);
                if (typeHandlers == null) {
                    //获取keyProperty的TypeHandler对象
                    typeHandlers = getTypeHandlers(typeHandlerRegistry, metaParam, keyProperties, rsmd);
                }
                //赋值到传入的参数
                populateKeys(rs, metaParam, keyProperties, typeHandlers);
            }
        }
    } catch (Exception e) {
        throw new ExecutorException("Error getting generated key or setting result to parameter object. Cause: " + e, e);
    } finally {
        if (rs != null) {
            try {
                rs.close();
            } catch (Exception e) {

            }
        }
    }
}
```

`getTypeHandlers()`方法会调用`typeHandlerRegistry.getTypeHandler()`方法根据`keyProperty`获取`TypeHandler`对象，方法实现比较简单，这里就不在详细分析了，下面来看`populateKeys()`方法，该方法会将获取到的值赋到用户传入的参数中

```java
private void populateKeys(ResultSet rs, MetaObject metaParam, String[] keyProperties, TypeHandler<?>[] typeHandlers) throws SQLException {
    //遍历keyProperty
    for (int i = 0; i < keyProperties.length; i++) {
        String property = keyProperties[i];
        //如果没有此字段的setter方法，则抛出异常
        if (!metaParam.hasSetter(property)) {
            throw new ExecutorException("No setter found for the keyProperty '" + property + "' in " + metaParam.getOriginalObject().getClass().getName() + ".");
        }
        TypeHandler<?> th = typeHandlers[i];
        if (th != null) {
            //获取ResultSet中的值
            Object value = th.getResult(rs, i + 1);
            //赋值
            metaParam.setValue(property, value);
        }
    }
}
```

#### SelectKeyGenerator

`SelectKeyGenerator`主要用来处理包含`selectKey `标签的`SQL`节点，主要字段如下

```java
/**
   * selectKey标签的order顺序，如果是BEFORE，则此变量为true
   * */
private boolean executeBefore;
/***
   * selectKey对应的MappedStatement对象
   * */
private MappedStatement keyStatement;
```

`processBefore()/processAfter()`两个方法会根据`executeBefore`的配置来决定是否执行

```java
public void processBefore(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    if (executeBefore) {
        processGeneratedKeys(executor, ms, parameter);
    }
}

public void processAfter(Executor executor, MappedStatement ms, Statement stmt, Object parameter) {
    if (!executeBefore) {
        processGeneratedKeys(executor, ms, parameter);
    }
}

```

`processGeneratedKeys()`会执行` selectKey`标签内的`SQL`，并赋值到用户传入的参数中

```java
private void processGeneratedKeys(Executor executor, MappedStatement ms, Object parameter) {
    try {
        if (parameter != null && keyStatement != null && keyStatement.getKeyProperties() != null) {
            //获取keyProperty
            String[] keyProperties = keyStatement.getKeyProperties();
            final Configuration configuration = ms.getConfiguration();
            //为参数创建MetaObject
            final MetaObject metaParam = configuration.newMetaObject(parameter);
            if (keyProperties != null) {
                //获取Executor执行器
                Executor keyExecutor = configuration.newExecutor(executor.getTransaction(), ExecutorType.SIMPLE);
                //执行selectKey标签中的SQL
                List<Object> values = keyExecutor.query(keyStatement, parameter, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER);
                //合法性校验
                if (values.size() == 0) {
                    throw new ExecutorException("SelectKey returned no data.");            
                } else if (values.size() > 1) {
                    throw new ExecutorException("SelectKey returned more than one value.");
                } else {
                    //为查询到的结果创建MetaObject对象
                    MetaObject metaResult = configuration.newMetaObject(values.get(0));
                    //如果keyProperty只配置了一个
                    if (keyProperties.length == 1) {

                        //如果keyProperty的名称可以和查询到的结果对应上，则通过MetaObject获取
                        if (metaResult.hasGetter(keyProperties[0])) {
                            setValue(metaParam, keyProperties[0], metaResult.getValue(keyProperties[0]));
                        } else {
                            //如果对应不上则直接获取结果值
                            setValue(metaParam, keyProperties[0], values.get(0));
                        }
                    } else {
                        //为多个keyProperty赋值
                        handleMultipleProperties(keyProperties, metaParam, metaResult);
                    }
                }
            }
        }
    } catch (ExecutorException e) {
        throw e;
    } catch (Exception e) {
        throw new ExecutorException("Error selecting key or setting result to parameter object. Cause: " + e, e);
    }
}
```

`handleMultipleProperties()`方法会为多个`keyProperty`进行赋值

```java
private void handleMultipleProperties(String[] keyProperties,
                                      MetaObject metaParam, MetaObject metaResult) {
    //获取所有的KeyColumn
    String[] keyColumns = keyStatement.getKeyColumns();

    if (keyColumns == null || keyColumns.length == 0) {
        // 如果没有KeyColumn，则使用keyProperty
        for (String keyProperty : keyProperties) {
            //赋值
            setValue(metaParam, keyProperty, metaResult.getValue(keyProperty));
        }
    } else {
        //如果数量对应不上，则抛出异常
        if (keyColumns.length != keyProperties.length) {
            throw new ExecutorException("If SelectKey has key columns, the number must match the number of key properties.");
        }
        //循环进行赋值
        for (int i = 0; i < keyProperties.length; i++) {
            setValue(metaParam, keyProperties[i], metaResult.getValue(keyColumns[i]));
        }
    }
}
```

`setValue()`方法会调用`MetaObject.setValue()`方法进行赋值，实现比较简单，这里就不在分析了。

