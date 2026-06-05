---
title: MyBatis源码分析(22)-核心处理层之简单结果集映射
tags:
  - MyBatis
date: 2019-04-13
categories:
 - MyBatis
---

# ResultSetHandler

进行数据库查询操作时，`Mybatis`会按照`mapper.xml`配置的映射规则(例如`resultMap`标签)将结果集映射为`Java`对象。这种映射机制是`Mybatis`核心功能之一。

在`statementHandler`执行完`SQL`语句后，会将结果交由`ResultSetHandler`进行映射处理，`ResultSetHandler`是一个接口，其中包含三个方法

```java
public interface ResultSetHandler {

    /**
   * 将结果集映射为java对象
   * **/
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;

    /**
   * 将结果集映射为游标对象返回
   * **/
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

    /**
   * 处理存储过程的输出参数
   * **/
    void handleOutputParameters(CallableStatement cs) throws SQLException;

}
```

`ResultSetHandler`唯一一个实现类为`DefaultResultSetHandler`，核心字段如下。用户也可以根据自己的需要自定义实现类

```java
/**
   * 关联的相关对象
   * */
private final Executor executor;
private final Configuration configuration;
private final MappedStatement mappedStatement;
private final RowBounds rowBounds;
private final ParameterHandler parameterHandler;
/**
   * 用户指定用于处理结果集的ResultHandler实现类
   * **/
private final ResultHandler<?> resultHandler;
```

## handleResultSets()

通过数据库查询得到的结果集会交由`handleResultSets()`方法进行处理，该方法也可以处理多结果集的情况，如以下的存储过程结果就会产生两个结果集

```sql
CREATE PROCEDURE test_multi_resultset()
BEGIN
select * from person;
select * from item;
END;
```

首先来看`handleResultSets()`的实现逻辑

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    //处理的结果会保存在此集合中
    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    //首先获取第一个结果集
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    //获取mapper.xml中定义的resultMap
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    //如果结果集不为空，则resultMap也不能为空
    validateResultMapsCount(rsw, resultMapCount);

    //遍历ResultMap集合
    while (rsw != null && resultMapCount > resultSetCount) {
        ResultMap resultMap = resultMaps.get(resultSetCount);
        //处理结果集
        //如果映射过程中发现引用了其它还未经过处理的ResultMap，则会将此ResultMap记录到nextResultMaps集合中，并跳过此属性，继续其它属性的映射
        handleResultSet(rsw, resultMap, multipleResults, null);
        //获取下一个resultSet
        rsw = getNextResultSet(stmt);
        //清空nestedResultObjects集合
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
    }
    
    //比如一个select标签中配置了多个resultSet，但是resultMap只配置了一个，后面会继续处理配置多结果集的情况

    //这里会处理多结果集，后面会详细分析，现在不需要太过于关注
    //获取sql语句查询后的所有结果集，该属性会给每个结果集一个名称
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
        while (rsw != null && resultSetCount < resultSets.length) {
            //根据resultSet名称获取ResultMapping对象
            ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
            if (parentMapping != null) {
                //获取外层的resultMap对象ID
                String nestedResultMapId = parentMapping.getNestedResultMapId();
                //根据外层对象ID获取resultMap
                ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                //处理结果集
                handleResultSet(rsw, resultMap, null, parentMapping);
            }
            //获取下一个结果集
            rsw = getNextResultSet(stmt);
            //清空nestedResultObjects集合
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
        }
    }

    //返回结果
    return collapseSingleResultList(multipleResults);
}
```

`getFirstResultSet()`和`getNextResultSet()`会调用`JDBC`本身的代码分别获取第一个结果集和下一个结果集

```java
private ResultSetWrapper getFirstResultSet(Statement stmt) throws SQLException {
    //获取结果集
    ResultSet rs = stmt.getResultSet();
    while (rs == null) {
        //防止未将第一个结果集返回，这里手动获取一次
        if (stmt.getMoreResults()) {
            rs = stmt.getResultSet();
        } else {
            if (stmt.getUpdateCount() == -1) {
                // 没有结果集
                break;
            }
        }
    }
    //将ResultSet封装为ResultSetWrapper返回
    return rs != null ? new ResultSetWrapper(rs, configuration) : null;
}
```

```java
private ResultSetWrapper getNextResultSet(Statement stmt) throws SQLException {
    try {
        //判断是否支持多个结果集
        if (stmt.getConnection().getMetaData().supportsMultipleResultSets()) {
            // 确认是否还有更多的结果集
            if (!((!stmt.getMoreResults()) && (stmt.getUpdateCount() == -1))) {
                //如果有则获取到，并封装为ResultSetWrapper返回
                ResultSet rs = stmt.getResultSet();
                return rs != null ? new ResultSetWrapper(rs, configuration) : null;
            }
        }
    } catch (Exception e) {

    }
    return null;
}
```

### ResultSetWrapper

`ResultSetWrapper`中封装了`resultSet`对象，记录了相关属性，并提供这些属性的相关方法，其主要属性如下

```java
/**
   * 绑定的ResultSet对象
   * */
private final ResultSet resultSet;
/**
   * TypeHandler工厂类
   * **/
private final TypeHandlerRegistry typeHandlerRegistry;
/**
   * 该结果集中的所有列名称
   * **/
private final List<String> columnNames = new ArrayList<String>();
/**
   * 该结果集中所有列的类型
   * **/
private final List<String> classNames = new ArrayList<String>();
/**
   * 该结果集中所有列的JdbcType
   * **/
private final List<JdbcType> jdbcTypes = new ArrayList<JdbcType>();
/**
   * 记录每列使用到的TypeHandler，key为列名，value是TypeHandler集合
   * **/
private final Map<String, Map<Class<?>, TypeHandler<?>>> typeHandlerMap = new HashMap<String, Map<Class<?>, TypeHandler<?>>>();
/**
   * 记录对应resultMap中匹配到的列，key为resultMap的ID
   * **/
private Map<String, List<String>> mappedColumnNamesMap = new HashMap<String, List<String>>();
/**
   * 记录对应resultMap中未匹配到的列，key为resultMap的ID
   * **/
private Map<String, List<String>> unMappedColumnNamesMap = new HashMap<String, List<String>>();
```

`ResultSetWrapper`的构造方法中会初始化`columnNames,jdbcTypes,classNames`三个集合的值，如下

```java
//获取resultSet的元信息
final ResultSetMetaData metaData = rs.getMetaData();
//获取列的数量
final int columnCount = metaData.getColumnCount();
for (int i = 1; i <= columnCount; i++) {
    //判断列名使用sql中的字段名还是"AS"关键字后面的名称
    columnNames.add(configuration.isUseColumnLabel() ? metaData.getColumnLabel(i) : metaData.getColumnName(i));
    //获取JdbcType
    jdbcTypes.add(JdbcType.forCode(metaData.getColumnType(i)));
    //获取对应的java类型
    classNames.add(metaData.getColumnClassName(i));
}
```

`ResultSetWrapper.getMappedColumnNames()`方法中，会返回指定`resultMap`中明确匹配的列信息，同时会初始化`mappedColumnNamesMap`和`unMappedColumnNamesMap`集合

```java
public List<String> getMappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    //根据resultMap获取被映射的列名
    List<String> mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    if (mappedColumnNames == null) {
        //如果为空，则调用方法进行加载
        loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
        //重新获取后返回
        mappedColumnNames = mappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    }
    return mappedColumnNames;
}
```

`loadMappedAndUnmappedColumnNames()`方法会判断每列是否被映射

```java
private void loadMappedAndUnmappedColumnNames(ResultMap resultMap, String columnPrefix) throws SQLException {
    //保存结果
    List<String> mappedColumnNames = new ArrayList<String>();
    List<String> unmappedColumnNames = new ArrayList<String>();
    //将列前缀转为大写
    final String upperColumnPrefix = columnPrefix == null ? null : columnPrefix.toUpperCase(Locale.ENGLISH);
    //为resultMap中的每个columns拼接前缀
    final Set<String> mappedColumns = prependPrefixes(resultMap.getMappedColumns(), upperColumnPrefix);
    //遍历该resultset中的所有列
    for (String columnName : columnNames) {
        //转为大写
        final String upperColumnName = columnName.toUpperCase(Locale.ENGLISH);
        //判断该列是否存在于resultMap的columns中，如果存在，则视为映射
        if (mappedColumns.contains(upperColumnName)) {
            mappedColumnNames.add(upperColumnName);
        } else {
            unmappedColumnNames.add(columnName);
        }
    }
    mappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), mappedColumnNames);
    unMappedColumnNamesMap.put(getMapKey(resultMap, columnPrefix), unmappedColumnNames);
}
```

### 简单映射

简单介绍`ResultSetWrapper`之后，下面继续分析`DefaultResultSetHandler.handleResultSets()`方法，该方法的核心是`handleResultSet()`，`handleResultSet()`会使用`resultMap`将`resultSet`映射为相应的对象并添加到`multipleResults`集合中

```java
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
        //当前ResultMap是否存在父节点，目前我们现在还没有涉及到此处的逻辑
        if (parentMapping != null) {
            handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
        } else {
            //用户是否自定义了resultHandler，如果没有则使用默认的DefaultResultHandler，当前为默认
            if (resultHandler == null) {
                DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
                handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
                multipleResults.add(defaultResultHandler.getResultList());
            } else {
                handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
            }
        }
    } finally {
        // 关闭resultSet
        closeResultSet(rsw.getResultSet());
    }
}
```

`handleRowValues()`为此方法的核心，`handleRowValues()`会判断嵌套映射还是简单映射，分别执行不同的逻辑

```java
public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    //判断是否有嵌套的ResultMap，此处逻辑我们后面会进行分析
    if (resultMap.hasNestedResultMaps()) {
        ensureNoRowBounds();
        checkResultHandler();
        handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
        //处理简单的映射
        handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
}
```

首先我们来分析简单映射，也就是`handleRowValuesForSimpleResultMap()`方法

```java
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
    throws SQLException {
    //创建上下文对象
    DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    //跳到结果集的指定行
    skipRows(rsw.getResultSet(), rowBounds);
    //判断是否还有记录并循环进行映射
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
        //解析discriminator标签，确定使用的resultMap
        ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
        //根据resultMap映射结果集中的一行记录
        Object rowValue = getRowValue(rsw, discriminatedResultMap);
        //保存映射后的结果
        storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
    }
}
```

下面我们来依次分析该方法涉及到的每个块内容

#### ResultContext&ResultHandler

`ResultContext`可以理解为映射结果操作中的"全局对象"，该类会记录已经映射成功的数量，并暂存映射结果，稍后会交由`ResultHandler`处理，`ResultContext`接口只有唯一的实现类`DefaultResultContext`，该类主要字段和方法如下

```java
/**
   * 暂存映射成功的一个对象
   * */
private T resultObject;
/**
   * 记录已经映射成功的数量
   * **/
private int resultCount;
/**
   * 控制是否停止自动映射
   * **/
private boolean stopped;

/**
   * 存储结果
   * **/
public void nextResultObject(T resultObject) {
    resultCount++;
    this.resultObject = resultObject;
}
```

`ResultHandler`接口只有一个方法`handleResult(ResultContext<? extends T> resultContext)`，处理结果集。相信看到这里大家应该会了解一些，`MyBatis`在映射成功后，会调用`DefaultResultContext.nextResultObject()`方法暂存结果集，随后将该对象作为参数调用`ResultHandler.handleResult()`处理结果集。`ResultHandler`接口有三个实现类，如下

![](/images/oss/%E5%8D%9A%E5%AE%A2/MyBatis/resultHandler%E7%B1%BB%E5%9B%BE.png)

经过上面的代码分析，这里用到的是`DefaultResultHandler`，我们这里先分析这一个实现类，该类实现比较简单，如下

```java
/**
   * 用于保存结果
   * **/
private final List<Object> list;
/**
   * 保存
   * **/
@Override
public void handleResult(ResultContext<? extends Object> context) {
    list.add(context.getResultObject());
}

/**
   * 返回结果
   * **/
public List<Object> getResultList() {
    return list;
}
```

#### skipRows()

记录回到`handleRowValuesForSimpleResultMap()`方法的分析，`skipRows()`方法用于跳转到`ResultSet`的指定行，实现如下

```java
private void skipRows(ResultSet rs, RowBounds rowBounds) throws SQLException {
    //查看结果集是否支持滚动，如支持则直接调用API
    if (rs.getType() != ResultSet.TYPE_FORWARD_ONLY) {
        if (rowBounds.getOffset() != RowBounds.NO_ROW_OFFSET) {
            rs.absolute(rowBounds.getOffset());
        }
    } else {
        //不支持滚动则通过next()方式定位到指定行
        for (int i = 0; i < rowBounds.getOffset(); i++) {
            rs.next();
        }
    }
}
```

#### shouldProcessMoreRows()

`shouldProcessMoreRows()`方法比较简单，使用`ResultContext`对象进行控制

```java
private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) throws SQLException {
    //根据ResultContext的相关属性进行控制
    return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
}
```

#### resolveDiscriminatedResultMap()

`resolveDiscriminatedResultMap()`方法用来解析`discriminator`标签，并确定最终使用的`resultMap`对象

```java
public ResultMap resolveDiscriminatedResultMap(ResultSet rs, ResultMap resultMap, String columnPrefix) throws SQLException {
    Set<String> pastDiscriminators = new HashSet<String>();
    //获取对象当前discriminator标签对应的Discriminator对象
    Discriminator discriminator = resultMap.getDiscriminator();
    while (discriminator != null) {
        //得到当前discriminator标签对应的值
        final Object value = getDiscriminatorValue(rs, discriminator, columnPrefix);
        //根据值查到ResultMapId
        final String discriminatedMapId = discriminator.getMapIdFor(String.valueOf(value));
        if (configuration.hasResultMap(discriminatedMapId)) {
            //获取ResultMap
            resultMap = configuration.getResultMap(discriminatedMapId);
            Discriminator lastDiscriminator = discriminator;
            discriminator = resultMap.getDiscriminator();
            //判断是否出现了环形引用，避免死循环
            if (discriminator == lastDiscriminator || !pastDiscriminators.add(discriminatedMapId)) {
                break;
            }
        } else {
            break;
        }
    }
    return resultMap;
}
```

其中`getDiscriminatorValue()`方法就是借助了`TypeHandler`获取相应的值，这里就不再叙述了，有兴趣可以自行分析

#### getRowValue()

前面说了那么多，其实都是在做一些前置的工作，而`getRowValue()`方法是`handleRowValuesForSimpleResultMap()`的核心方法，用来获取一行的数据

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap) throws SQLException {
    //此集合与延迟加载有关，后面再进行介绍
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    //根据resultMap映射出对象，对象的类型由ResultMap的type属性指定
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, null);
    //判断映射类的有效性
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        //为映射类创建MetaObject对象
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        //记录是否使用构造方法创建了对象
        boolean foundValues = this.useConstructorMappings;
        //是否开启了自动映射
        if (shouldApplyAutomaticMappings(resultMap, false)) {
            foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, null) || foundValues;
        }
        //为映射类赋值
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, null) || foundValues;
        foundValues = lazyLoader.size() > 0 || foundValues;
        //如果foundValues为false，则根据配置的returnInstanceForEmptyRow属性决定返回null
        rowValue = (foundValues || configuration.isReturnInstanceForEmptyRow()) ? rowValue : null;
    }
    return rowValue;
}
```

##### createResultObject()

`createResultObject()`负责创建映射类

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, ResultLoaderMap lazyLoader, String columnPrefix) throws SQLException {
    //重置该属性
    this.useConstructorMappings = false;
    //保存构造方法的参数类型
    final List<Class<?>> constructorArgTypes = new ArrayList<Class<?>>();
    //保存构造方法的参数
    final List<Object> constructorArgs = new ArrayList<Object>();
    //调用重载方法创建映射对象
    Object resultObject = createResultObject(rsw, resultMap, constructorArgTypes, constructorArgs, columnPrefix);
    //检测映射结果的有效性
    if (resultObject != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        //获取所有的property节点
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        //遍历
        for (ResultMapping propertyMapping : propertyMappings) {
            // 如果存在嵌套查询，则创建出嵌套对象
            if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
                resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
                break;
            }
        }
    }
    //记录当前的映射结果
    this.useConstructorMappings = (resultObject != null && !constructorArgTypes.isEmpty()); 
    return resultObject;
}
```

`createResultObject()`的重载方法是创建映射对象的核心逻辑，该方法会根据不同的情况使用不同的方式来创建映射对象

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
    throws SQLException {
    //获取ResultMap的对应类，该属性由ResultMap的type属性决定
    final Class<?> resultType = resultMap.getType();
    //为该类创建MetaClass对象
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    //获取contructor节点
    final List<ResultMapping> constructorMappings = resultMap.getConstructorResultMappings();
    //情况1：结果只有一列，且TypeHandler可以处理
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
        //首先查找对应的TypeHandler对象，其次使用TypeHandler对象将该列转换为Java对象
        return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    } else if (!constructorMappings.isEmpty()) {
        //情况2：如果resultMap标签中包含contructor节点
        //根据contructor节点的配置创建对象
        return createParameterizedResultObject(rsw, resultType, constructorMappings, constructorArgTypes, constructorArgs, columnPrefix);
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
        //情况3：如果是接口或者有默认的构造方法
        //直接创建对象
        return objectFactory.create(resultType);
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
        //情况4：如果开启了自动映射
        //查找合适的构造方法创建对象
        return createByConstructorSignature(rsw, resultType, constructorArgTypes, constructorArgs, columnPrefix);
    }
    //如果都不符合，则抛出异常
    throw new ExecutorException("Do not know how to create an instance of " + resultType);
}
```

###### createPrimitiveResultObject()

`createPrimitiveResultObject()`方法会查找对应的`TypeHandler`进行处理

```java
private Object createPrimitiveResultObject(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final Class<?> resultType = resultMap.getType();
    final String columnName;
    //如果没有配置节点，则获取ResultSet的第一列
    if (!resultMap.getResultMappings().isEmpty()) {
        final List<ResultMapping> resultMappingList = resultMap.getResultMappings();
        //注意，这里只会处理第一列
        final ResultMapping mapping = resultMappingList.get(0);
        columnName = prependPrefix(mapping.getColumn(), columnPrefix);
    } else {
        columnName = rsw.getColumnNames().get(0);
    }
    //查找对应的TypeHandler对象
    final TypeHandler<?> typeHandler = rsw.getTypeHandler(resultType, columnName);
    //获取结果
    return typeHandler.getResult(rsw.getResultSet(), columnName);
}
```

###### createParameterizedResultObject()

`createParameterizedResultObject()`会根据`contructor`节点的配置创建对象

```java
Object createParameterizedResultObject(ResultSetWrapper rsw, Class<?> resultType, List<ResultMapping> constructorMappings,
                                       List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix) {
    boolean foundValues = false;
    //遍历contructor节点
    for (ResultMapping constructorMapping : constructorMappings) {
        //获取该节点的java类型
        final Class<?> parameterType = constructorMapping.getJavaType();
        //获取该节点的列名
        final String column = constructorMapping.getColumn();
        final Object value;
        try {
            //处理嵌套查询
            if (constructorMapping.getNestedQueryId() != null) {
                value = getNestedQueryConstructorValue(rsw.getResultSet(), constructorMapping, columnPrefix);
            } else if (constructorMapping.getNestedResultMapId() != null) {
                //处理嵌套映射
                final ResultMap resultMap = configuration.getResultMap(constructorMapping.getNestedResultMapId());
                value = getRowValue(rsw, resultMap);
            } else {
                //查找对应的TypeHandler获取值
                final TypeHandler<?> typeHandler = constructorMapping.getTypeHandler();
                value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(column, columnPrefix));
            }
        } catch (ResultMapException e) {
            throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
        } catch (SQLException e) {
            throw new ExecutorException("Could not process result for mapping: " + constructorMapping, e);
        }
        //记录列类型和列值
        constructorArgTypes.add(parameterType);
        constructorArgs.add(value);
        foundValues = value != null || foundValues;
    }
    //返回结果
    return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
}
```

###### createByConstructorSignature()

`createByConstructorSignature()`会根据`ResultSet`查找合适的构造方法来创建对象

```java
private Object createByConstructorSignature(ResultSetWrapper rsw, Class<?> resultType, List<Class<?>> constructorArgTypes, List<Object> constructorArgs,
                                            String columnPrefix) throws SQLException {
    //获取类中的所有构造方法
    for (Constructor<?> constructor : resultType.getDeclaredConstructors()) {
        //如果构造方法的参数与ResultSet中的列一一对应
        if (typeNames(constructor.getParameterTypes()).equals(rsw.getClassNames())) {
            boolean foundValues = false;
            //循环构造方法的各个参数，并获取值加入到constructorArgs集合中
            for (int i = 0; i < constructor.getParameterTypes().length; i++) {
                //获取参数类型
                Class<?> parameterType = constructor.getParameterTypes()[i];
                //获取ResultSet的列名
                String columnName = rsw.getColumnNames().get(i);
                //根据列名和类型查找TypeHandler对象
                TypeHandler<?> typeHandler = rsw.getTypeHandler(parameterType, columnName);
                //获取结果并记录到集合中
                Object value = typeHandler.getResult(rsw.getResultSet(), prependPrefix(columnName, columnPrefix));
                constructorArgTypes.add(parameterType);
                constructorArgs.add(value);
                foundValues = value != null || foundValues;
            }
            //创建对象
            return foundValues ? objectFactory.create(resultType, constructorArgTypes, constructorArgs) : null;
        }
    }
    throw new ExecutorException("No constructor found in " + resultType.getName() + " matching " + rsw.getClassNames());
}
```

至此，类已经被创建出来了，我们继续回到`getRowValue()`方法中

##### shouldApplyAutomaticMappings()

`shouldApplyAutomaticMappings()`方法会判断是否开启了自动映射，判断条件有两个，一是查看`ResultMap`标签中的`autoMapping`属性，二是查看`mybatis-config.xml`中的`autoMappingBehavior`配置。方法比较简单，这里就不再阐述了

##### applyAutomaticMappings()

`applyAutomaticMappings()`方法会映射未在`ResultMap`标签中配置映射的字段

```java
private boolean applyAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    //获取在ResultMap标签中没有配置，但是在ResultSet存在的列，并封装为UnMappedColumnAutoMapping对象
    List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    boolean foundValues = false;
    if (autoMapping.size() > 0) {
        for (UnMappedColumnAutoMapping mapping : autoMapping) {
            //获取该列的值
            final Object value = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
            if (value != null) {
                foundValues = true;
            }
            if (value != null || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
                // 赋值
                metaObject.setValue(mapping.property, value);
            }
        }
    }
    return foundValues;
}

```

###### createAutomaticMappings()

`createAutomaticMappings()`方法会获取未映射的字段集合，并封装成`UnMappedColumnAutoMapping`集合返回

```java
private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    //首先尝试在缓存中获取
    final String mapKey = resultMap.getId() + ":" + columnPrefix;
    List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
    if (autoMapping == null) {
        autoMapping = new ArrayList<UnMappedColumnAutoMapping>();
        //从ResultSetWrapper获取未映射的列名
        final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
        for (String columnName : unmappedColumnNames) {
            String propertyName = columnName;
            if (columnPrefix != null && !columnPrefix.isEmpty()) {
                //去掉列前缀，获取字段名称
                if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
                    propertyName = columnName.substring(columnPrefix.length());
                } else {
                    continue;
                }
            }
            //获取字段值
            final String property = metaObject.findProperty(propertyName, configuration.isMapUnderscoreToCamelCase());
            if (property != null && metaObject.hasSetter(property)) {
                final Class<?> propertyType = metaObject.getSetterType(property);
                if (typeHandlerRegistry.hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
                    final TypeHandler<?> typeHandler = rsw.getTypeHandler(propertyType, columnName);
                    autoMapping.add(new UnMappedColumnAutoMapping(columnName, property, typeHandler, propertyType.isPrimitive()));
                } else {
                    configuration.getAutoMappingUnknownColumnBehavior()
                        .doAction(mappedStatement, columnName, property, propertyType);
                }
            } else{
                configuration.getAutoMappingUnknownColumnBehavior()
                    .doAction(mappedStatement, columnName, (property != null) ? property : propertyName, null);
            }
        }
        //加入缓存
        autoMappingsCache.put(mapKey, autoMapping);
    }
    return autoMapping;
}
```

##### applyPropertyMappings()

`applyPropertyMappings()`方法负责为明确指定了映射关系的字段赋值

```java
private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
    throws SQLException {
    //获取所有匹配到映射关系的列
    final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
    boolean foundValues = false;
    //获取property标签对应的ResultMapping对象
    final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
    for (ResultMapping propertyMapping : propertyMappings) {
        //拼接前缀
        String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        if (propertyMapping.getNestedResultMapId() != null) {
            // 如果需要使用一个嵌套的ResultMap，则忽略此列
            column = null;
        }
        //校验有效性
        if (propertyMapping.isCompositeResult()
            || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
            || propertyMapping.getResultSet() != null) {
            //获取对应的值
            Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
            final String property = propertyMapping.getProperty();
            if (property == null) {
                continue;
            } else if (value == DEFERED) {
                //DEFERED为占位符，后面会详细介绍
                foundValues = true;
                continue;
            }
            if (value != null) {
                foundValues = true;
            }
            if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
                //赋值
                metaObject.setValue(property, value);
            }
        }
    }
    return foundValues;
}
```

###### getPropertyMappingValue()

`getPropertyMappingValue()`方法负责映射操作

```java
private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
    throws SQLException {
    //嵌套查询，后面会详细介绍
    if (propertyMapping.getNestedQueryId() != null) {
        return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
        //多结果集的处理，后面会详细介绍
        addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
        return DEFERED;
    } else {
        //使用TypeHandler获取值
        final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
        final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        return typeHandler.getResult(rs, column);
    }
}
```



##### storeObject()

`storeObject()`用来保存映射后的结果

```java
private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    //根据是否存在父类ResultMapping，分别执行不同的方法
    if (parentMapping != null) {
        //嵌套映射的过程我们后面进行讲解
        linkToParents(rs, parentMapping, rowValue);
    } else {
        callResultHandler(resultHandler, resultContext, rowValue);
    }
}
```

###### callResultHandler()

`callResultHandler()`方法的实现比较简单，直接调用`ResultHandler.handleResult()`方法处理结果对象

```java
private void callResultHandler(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue) {
    //暂存结果对象
    resultContext.nextResultObject(rowValue);
    ((ResultHandler<Object>)resultHandler).handleResult(resultContext);
}
```

至此，我们的整个简单映射过程就已经完成了，后面的文章会继续分析嵌套映射的情况。