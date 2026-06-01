---
title: MyBatis源码分析(23)-核心处理层之嵌套映射
tags:
  - MyBatis
date: 2019-04-14
categories:
 - MyBatis
---

### 嵌套映射

前面我们讲到了`MyBatis`的简单结果集映射，这篇文章主要分析嵌套映射的过程

在实际使用中，很大可能会遇到多表联查的情况，这样就需要在`ResultMap`标签中配置多对一或一对多的的映射关系，上篇文章我们讲到，`MyBatis`处理简单映射和嵌套映射的判断是在`handleRowValues()`方法中，其中，嵌套映射的分支调用的是`handleRowValuesForNestedResultMaps()`方法，在映射开始时，会为每行记录创建一个独一无二的值，该值用来标识唯一的结果对象

```java
private void handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    //创建结果集"全局对象"
    final DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
    //跳转到指定行，此方法之前已经分析过
    skipRows(rsw.getResultSet(), rowBounds);
    Object rowValue = previousRowValue;
    //是否还有数据需要映射
    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
        //根据discriminated标签决定最后要使用的resultMap
        final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
        //为当前行创建CacheKey，该值不仅被用在缓存中，也会标识此行的结果对象
        final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
        //查询该行是否被映射过
        Object partialObject = nestedResultObjects.get(rowKey);
        //查询resultOrdered的值，
        // 如果该值为true，会创建一个新的对象，则不会引用nestedResultObjects缓存中的值，这样可以提前释放内存
        if (mappedStatement.isResultOrdered()) {
            if (partialObject == null && rowValue != null) {
                nestedResultObjects.clear();
                //保存对象
                storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
            }
            //映射单行记录
            rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
        } else {
            rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
            if (partialObject == null) {
                storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
            }
        }
    }
    //对resultOrdered为true时的特殊处理
    if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
        storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
        previousRowValue = null;
    } else if (rowValue != null) {
        previousRowValue = rowValue;
    }
}
```

#### createRowKey()

`createRowKey()`负责为对应的结果对象创建唯一的标识，此方法会分为几种情况，如下

```java
private CacheKey createRowKey(ResultMap resultMap, ResultSetWrapper rsw, String columnPrefix) throws SQLException {
    //创建CacheKey对象
    final CacheKey cacheKey = new CacheKey();
    //拼接ResultMap的ID
    cacheKey.update(resultMap.getId());
    //获取ID节点，如果ID节点为空则会获取property节点
    List<ResultMapping> resultMappings = getResultMappingsForRowKey(resultMap);
    //如果一个节点也没获取到则分为两种情况
    //1、如果结果集是Map集合，则会拼接ResultSet中的所有列名与列值
    //2、如果不是Map集合，则会拼接ResultSetWrapper对象中未匹配的列名和列值
    if (resultMappings.size() == 0) {
        if (Map.class.isAssignableFrom(resultMap.getType())) {
            createRowKeyForMap(rsw, cacheKey);
        } else {
            createRowKeyForUnmappedProperties(resultMap, rsw, cacheKey, columnPrefix);
        }
    } else {
        //只要找到了一个节点，则拼接此节点的列名与列值
        createRowKeyForMappedProperties(resultMap, rsw, cacheKey, resultMappings, columnPrefix);
    }
    //如果参与计算的数量小于2，则返回空对象
    if (cacheKey.getUpdateCount() < 2) {
        return CacheKey.NULL_CACHE_KEY;
    }    
    return cacheKey;
}
```

`createRowKeyForMap()、createRowKeyForUnmappedProperties()、createRowKeyForMappedProperties()`三个方法实现比较简单，这里就不再分析了

#### getRowValue()

`getRowValue()`方法是嵌套映射的核心逻辑，该方法的重载会映射对应行，并将映射到的结果赋值到"*所属对象*"中，此处说的"*所属对象*"并不是继承的父类，而是类似于一对多中“一”的一方，举个例子，比如下面的类(一篇文章会有多个评论)，这里，我们把`Article`对象叫做`Comment`对象的"*所属对象*"

```java
public class Article {
    /**
     * 文章标题
     * */
    private String title;
    /**
     * 文章评论
     * */
    private List<Comment> comments;
}
```

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, CacheKey combinedKey, String columnPrefix, Object partialObject) throws SQLException {
    //获取ResultMap的ID
    final String resultMapId = resultMap.getId();
    //获取所属对象
    Object rowValue = partialObject;
    //如果所属对象不为空
    if (rowValue != null) {
        //为所属对象创建MetaObject对象
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        //将此对象加入到ancestorObjects集合中暂存，便与其它方法使用
        putAncestor(rowValue, resultMapId, columnPrefix);
        //开始处理嵌套查询
        applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
        //移除
        ancestorObjects.remove(resultMapId);
    } else {
        //如果所属对象为空
        //此类涉及到延迟加载，后面会进行分析
        final ResultLoaderMap lazyLoader = new ResultLoaderMap();
        //创建该行的java对象
        rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
        if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
            //为映射出的对象创建MetaObject
            final MetaObject metaObject = configuration.newMetaObject(rowValue);
            boolean foundValues = this.useConstructorMappings;
            //自动映射逻辑
            if (shouldApplyAutomaticMappings(resultMap, true)) {
                foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
            }
            //对已经明确指定映射关系的属性赋值
            foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
            //加入到集合中
            putAncestor(rowValue, resultMapId, columnPrefix);
            //处理嵌套查询
            foundValues = applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, true) || foundValues;
            ancestorObjects.remove(resultMapId);
            foundValues = lazyLoader.size() > 0 || foundValues;
            rowValue = (foundValues || configuration.isReturnInstanceForEmptyRow()) ? rowValue : null;
        }
        //加入到缓存中
        if (combinedKey != CacheKey.NULL_CACHE_KEY) {
            nestedResultObjects.put(combinedKey, rowValue);
        }
    }
    return rowValue;
}
```

##### applyNestedResultMappings()

`applyNestedResultMappings()`负责处理`ResultMap`标签中每个节点的嵌套映射

```java
private boolean applyNestedResultMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String parentPrefix, CacheKey parentRowKey, boolean newObject) {
    boolean foundValues = false;
    //遍历并获取ResultMap中每个节点
    for (ResultMapping resultMapping : resultMap.getPropertyResultMappings()) {
        //判断是否存在嵌套映射
        //需要注意的是。即使嵌套映射标签中没有添加ResultMapId属性
        //MyBatis初始化时也会自动添加
        final String nestedResultMapId = resultMapping.getNestedResultMapId();
        if (nestedResultMapId != null && resultMapping.getResultSet() == null) {
            try {
                //获取列前缀
                final String columnPrefix = getColumnPrefix(parentPrefix, resultMapping);
                //获取嵌套的ResultMap
                final ResultMap nestedResultMap = getNestedResultMap(rsw.getResultSet(), nestedResultMapId, columnPrefix);
                if (resultMapping.getColumnPrefix() == null) {
                    //根据ResultMapId的值获取相关对象，如果不为空则证明发生了循环引用
                    // 直接关联到所属对象，不需要再次新建对象
                    Object ancestorObject = ancestorObjects.get(nestedResultMapId);
                    if (ancestorObject != null) {
                        if (newObject) {
                            //关联到所属对象
                            linkObjects(metaObject, resultMapping, ancestorObject);
                        }
                        continue;
                    }
                }
                //为该行数据创建CacheKey
                final CacheKey rowKey = createRowKey(nestedResultMap, rsw, columnPrefix);
                //绑定所属对象的cacheKey，此操作的目的是确保不同对象内的同一个属性引用到的是不用的对象
                //如article1->(comment1,comment2),article2->(comment2)
                //我们希望此处的两个comment2是两个不同的对象，而不是同一个对象
                final CacheKey combinedKey = combineKeys(rowKey, parentRowKey);
                Object rowValue = nestedResultObjects.get(combinedKey);
                boolean knownValue = (rowValue != null);
                //处理集合类型的值
                instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject); // mandatory
                if (anyNotNullColumnHasValue(resultMapping, columnPrefix, rsw)) {
                    //将结果集的对应行映射为对象
                    rowValue = getRowValue(rsw, nestedResultMap, combinedKey, columnPrefix, rowValue);
                    if (rowValue != null && !knownValue) {
                        //关联到所属对象
                        linkObjects(metaObject, resultMapping, rowValue);
                        foundValues = true;
                    }
                }
            } catch (SQLException e) {
                throw new ExecutorException("Error getting nested result map values for '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
            }
        }
    }
    return foundValues;
}
```

`instantiateCollectionPropertyIfAppropriate()、linkObjects()`比较简单，这里就不再描述了

#### storeObject()

前一篇文章介绍`storeObject()`方法时，还有一个分支没有讲解，`linkToParents()`方法会为结果集创建`CacheKey`并且调用`linkObjects()`方法关联到所属对象中，实现比较简单，这里也不在叙述了

至此，`MyBatis`中对于简单结果集和嵌套结果集的映射我们都已经分析完成了，对于其中涉及的延迟加载等策略，后面的文章会进行分析