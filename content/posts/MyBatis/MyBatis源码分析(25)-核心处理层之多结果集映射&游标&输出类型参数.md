---
title: MyBatis源码分析(25)-核心处理层之多结果集映射&游标&输出类型参数
tags:
  - MyBatis
date: 2019-04-21
categories:
 - MyBatis
---

前面已经分析过了简单结果集映射、嵌套映射、延迟加载、嵌套查询等，这篇文章主要是对`DefaultResultSetHandler.handleResultSets()`方法的查缺补漏，首先是多结果集映射

#### 多结果集映射

前面分析`DefaultResultSetHandler.handleResultSets()`方法时，我们提到如果遇到多个结果集，并且在解析第一个结果集时发现其对应的`ResultMap`还未进行解析，则会添加到`nextResultMaps()`集合中后续进行解析，该操作主要是在`DefaultResultSetHandler.getPropertyMappingValue()`进行，相关代码如下

```java
//嵌套查询，后面会详细介绍
if (propertyMapping.getNestedQueryId() != null) {
    //忽略..
} else if (propertyMapping.getResultSet() != null) {
    //多结果集的处理，后面会详细介绍
    addPendingChildRelation(rs, metaResultObject, propertyMapping);   
    return DEFERED;
} else {
    //忽略..
}
```

`addPendingChildRelation()`方法实现如下

```java
private void addPendingChildRelation(ResultSet rs, MetaObject metaResultObject, ResultMapping parentMapping) throws SQLException {
    //创建CacheKey对象，主要由三部分组成
    //1、parentMapping对象，也就是所属节点
    //2、parentMapping中的列名
    //3、parentMapping每列的实际值
    CacheKey cacheKey = createKeyForMultipleResults(rs, parentMapping, parentMapping.getColumn(), parentMapping.getColumn());
    //创建PendingRelation对象
    //PendingRelation对象无其他方法，只有两个公共字段metaObject和propertyMapping
    PendingRelation deferLoad = new PendingRelation();
    deferLoad.metaObject = metaResultObject;
    deferLoad.propertyMapping = parentMapping;
    //查看缓存中是否存在，如不存在则放入缓存
    List<PendingRelation> relations = pendingRelations.get(cacheKey);
    if (relations == null) {
        relations = new ArrayList<DefaultResultSetHandler.PendingRelation>();
        pendingRelations.put(cacheKey, relations);
    }
    relations.add(deferLoad);
    //如果此resultSet还没有经过解析，则放入nextResultMaps
    ResultMapping previous = nextResultMaps.get(parentMapping.getResultSet());
    if (previous == null) {
        nextResultMaps.put(parentMapping.getResultSet(), parentMapping);
    } else {
        //如果根据resultset获取到了别的ResultMapping则抛出异常
        if (!previous.equals(parentMapping)) {
            throw new ExecutorException("Two different properties are mapped to the same resultSet");
        }
    }
}
```

至此，`ResultSetHandler.handleResultSets()`就已经分析完成了，下面分析剩余的两个方法

#### 游标

`ResultSetHandler.handleCursorResultSets()`负责将结果集映射为游标对象返回，该方法返回值为`<E> Cursor<E>` ，`Cursor`是一个接口，继承了`Closeable, Iterable`两个类

`DefaultResultSetHandler.handleCursorResultSets()`方法会将`resultSet`和`resultMap`封装成`DefaultCursor`对象返回

```java
public <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling cursor results").object(mappedStatement.getId());

    //获取第一个结果集
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    //获取所有的ResultMap
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();

    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    //游标对象只能匹配一个ResultMap对象
    if (resultMapCount != 1) {
        throw new ExecutorException("Cursor results cannot be mapped to multiple resultMaps");
    }

    ResultMap resultMap = resultMaps.get(0);
    //封装成DefaultCursor对象返回
    return new DefaultCursor<E>(this, resultMap, rsw, rowBounds);
}
```

`DefaultCursor`实现了`Cursor`接口，其主要字段如下

```java
// 对应的对象，前面分析过
private final DefaultResultSetHandler resultSetHandler;
private final ResultMap resultMap;
private final ResultSetWrapper rsw;
private final RowBounds rowBounds;
//ObjectWrapperResultHandler实现了ResultHandler接口，用来保存处理后的结果对象
private final ObjectWrapperResultHandler<T> objectWrapperResultHandler = new ObjectWrapperResultHandler<T>();

//获取结果对象的迭代器
private final CursorIterator cursorIterator = new CursorIterator();
//是否正在迭代结果集
private boolean iteratorRetrieved;

//当前游标的状态
private CursorStatus status = CursorStatus.CREATED;
//记录已经完成映射的行数
private int indexWithRowBound = -1;

private enum CursorStatus {

    /**
         * 新创建的游标，数据库ResultSet消耗尚未开始
         */
    CREATED,
    /**
         * 当前正在使用的游标，数据库ResultSet消耗已经开始
         */
    OPEN,
    /**
         * 关闭的游标，没有完全消耗
         */
    CLOSED,
    /**
         * 完全消耗的游标，消耗的游标总是关闭
         */
    CONSUMED
}
```

`DefaultCursor.iterator()`方法会返回`CursorIterator`对象，`CursorIterator`是`DefaultCursor`的内部类，实现了`Iterator`接口，其中比较核心逻辑都是在此类中进行调用

既然实现了`Iterator`接口，那核心逻辑就一定是`hasNext()/next()`方法

```java
@Override
public boolean hasNext() {
    //调用DefaultCursor.fetchNextUsingRowBound()方法映射下一条记录
    if (object == null) {
        object = fetchNextUsingRowBound();
    }
    return object != null;
}

@Override
public T next() {
    // 获取hasNext中提取的对象
    T next = object;

    //如果为空则继续获取下一个
    if (next == null) {
        next = fetchNextUsingRowBound();
    }

    //如果不为空，记录iteratorIndex并返回结果
    if (next != null) {
        object = null;
        iteratorIndex++;
        return next;
    }
    throw new NoSuchElementException();
}
```

`fetchNextObjectFromDatabase()`方法负责定位到对应的结果行

```java
protected T fetchNextUsingRowBound() {
    //首先映射一条记录
    T result = fetchNextObjectFromDatabase();
    //跳过rowBounds中的offset行
    while (result != null && indexWithRowBound < rowBounds.getOffset()) {
        result = fetchNextObjectFromDatabase();
    }
    //返回结果
    return result;
}
```

`fetchNextObjectFromDatabase()`负责映射对应行的记录

```java
protected T fetchNextObjectFromDatabase() {
    //如果当前已经处于关闭状态，直接返回null
    if (isClosed()) {
        return null;
    }

    try {
        //标记为打开状态
        status = CursorStatus.OPEN;
        //映射当前行
        resultSetHandler.handleRowValues(rsw, resultMap, objectWrapperResultHandler, RowBounds.DEFAULT, null);
    } catch (SQLException e) {
        throw new RuntimeException(e);
    }
    //获取映射后的结果
    T next = objectWrapperResultHandler.result;
    if (next != null) {
        //增加indexWithRowBound
        indexWithRowBound++;
    }
    // 如果都已经处理完毕了，则关闭此ResultSet，Statement等对象并标记为销毁状态
    if (next == null || (getReadItemsCount() == rowBounds.getOffset() + rowBounds.getLimit())) {
        close();
        status = CursorStatus.CONSUMED;
    }
    objectWrapperResultHandler.result = null;

    return next;
}
```

可以看到，游标对象是每次获得一行数据时才会去映射，而不是直接映射

#### 输出参数

`DefaultResultSetHandler.handleOutputParameters()`负责处理存储过程的输出参数

```java
public void handleOutputParameters(CallableStatement cs) throws SQLException {
    //获取用户传入的实参
    final Object parameterObject = parameterHandler.getParameterObject();
    //创建MetaObject对象
    final MetaObject metaParam = configuration.newMetaObject(parameterObject);
    //获取SQL语句中的参数集合
    final List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    for (int i = 0; i < parameterMappings.size(); i++) {
        final ParameterMapping parameterMapping = parameterMappings.get(i);
        //如果是输出型参数
        if (parameterMapping.getMode() == ParameterMode.OUT || parameterMapping.getMode() == ParameterMode.INOUT) {
            //如果是ResultSet类型的参数，则需要进行映射
            if (ResultSet.class.equals(parameterMapping.getJavaType())) {
                handleRefCursorOutputParameter((ResultSet) cs.getObject(i + 1), parameterMapping, metaParam);
            } else {
                //直接调用TypeHandler进行处理
                final TypeHandler<?> typeHandler = parameterMapping.getTypeHandler();
                metaParam.setValue(parameterMapping.getProperty(), typeHandler.getResult(cs, i + 1));
            }
        }
    }
}
```

`handleRefCursorOutputParameter()`会根据指定的`ResultMap`映射`resultSet`

```java
private void handleRefCursorOutputParameter(ResultSet rs, ParameterMapping parameterMapping, MetaObject metaParam) throws SQLException {
    if (rs == null) {
        return;
    }
    try {
        //获取对应的ResultMap对象
        final String resultMapId = parameterMapping.getResultMapId();
        final ResultMap resultMap = configuration.getResultMap(resultMapId);
        final DefaultResultHandler resultHandler = new DefaultResultHandler(objectFactory);
        final ResultSetWrapper rsw = new ResultSetWrapper(rs, configuration);
        //映射结果
        handleRowValues(rsw, resultMap, resultHandler, new RowBounds(), null);
        //存储结果
        metaParam.setValue(parameterMapping.getProperty(), resultHandler.getResultList());
    } finally {
        //关闭ResultSet对象
        closeResultSet(rs);
    }
}
```



至此，`DefaultResultSetHandler`的原理就已经分析完成了