---
title: MyBatis源码分析(18)-核心处理层之mapper解析(1)
tags:
  - MyBatis
date: 2019-02-23
categories:
 - MyBatis
---

## mapper.xml解析

上篇文章中我们介绍了`Mybatis`解析配置文件的过程，今天主要了解`mapper`文件的解析过程。该工作主要由`XMLMapperBuilder.parse()`方法完成。

### XMLMapperBuilder

`XMLMapperBuilder`负责解析`mapper`文件，首先了解一下主要属性

```java
//对应mapper文件的parser对象
private XPathParser parser;
//每一个mapper文件都对应一个MapperBuilderAssistant对象
//其中记录了当前的namespace，当前使用的缓存对象等
private MapperBuilderAssistant builderAssistant;
//用来存储<sql>节点，也就是存储sql片段
private Map<String, XNode> sqlFragments;
//该类的唯一标识，也是mapper文件的地址
private String resource;
```



`XMLMapperBuilder.parse()`方法实现与解析配置文件时类似，调用不同的方法解析不同的内容。

```java
public void parse() {
    //是否已经加载过了此文件
    if (!configuration.isResourceLoaded(resource)) {
        //解析mapper
        configurationElement(parser.evalNode("/mapper"));
        //记录已经解析过的文件
        configuration.addLoadedResource(resource);
        //绑定mapper接口
        bindMapperForNamespace();
    }

    //处理configurationElement中解析失败的ResultMap标签
    parsePendingResultMaps();
    //处理configurationElement中解析失败的CacheRefs标签
    parsePendingChacheRefs();
    //处理configurationElement中解析失败的SQL语句
    parsePendingStatements();
}
```

#### configurationElement()

`configurationElement()`方法负责解析`mapper`文件

```java
private void configurationElement(XNode context) {
    try {
        //取出当前文件的namespace
        String namespace = context.getStringAttribute("namespace");
        if (namespace == null || namespace.equals("")) {
            throw new BuilderException("....");
        }
        //记录当前的namespace
        builderAssistant.setCurrentNamespace(namespace);
        //解析<cache-ref>节点
        cacheRefElement(context.evalNode("cache-ref"));
        //解析<cache>节点
        cacheElement(context.evalNode("cache"));
        //解析<parameterMap>节点,已被废弃,故不再叙述
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        //解析<resultMap>节点
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        //解析<sql>节点
        sqlElement(context.evalNodes("/mapper/sql"));
        //解析select|insert|update|delete等SQL节点
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("...");
    }
}
```

##### 解析&lt;cache-ref&gt;节点

`cacheRefElement()`方法负责解析&lt;cache-ref&gt;节点，方式实现如下

```java
private void cacheRefElement(XNode context) {
    //判断该节点是否为空
    if (context != null) {
        //记录当前的namespace和引用缓存的namespace对应关系
        configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
        //创建解析对象
        CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
        try {
            //开始解析
            cacheRefResolver.resolveCacheRef();
        } catch (IncompleteElementException e) {
            //如果发生异常则暂时记录，稍后再次进行解析
            configuration.addIncompleteCacheRef(cacheRefResolver);
        }
    }
}
```

`CacheRefResolver.resolveCacheRef()`方法会调用`MapperBuilderAssistant.useCacheRef()`方法，该方法实现如下

````java
public Cache useCacheRef(String namespace) {
    //校验namespace
    if (namespace == null) {
        throw new BuilderException("....");
    }
    try {
        //解析标识
        unresolvedCacheRef = true;
        Cache cache = configuration.getCache(namespace);
        //如果没有找到cache对象则抛出异常
        if (cache == null) {
            throw new IncompleteElementException("....");
        }
        //将引用的cache对象与当前cache绑定
        currentCache = cache;
        //修改标识
        unresolvedCacheRef = false;
        return cache;
    } catch (IllegalArgumentException e) {
        throw new IncompleteElementException("...");
    }
}
````

##### 解析&lt;cache&gt;节点

`cacheElement()`方法负责解析&lt;cache&gt;节点，方式实现如下

```java
private void cacheElement(XNode context) throws Exception {
    if (context != null) {
        //获取cache对象的类型，默认为PERPETUAL
        String type = context.getStringAttribute("type", "PERPETUAL");
        Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
        //获取cache对象的eviction属性，默认是lru
        String eviction = context.getStringAttribute("eviction", "LRU");
        Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
        //获取其它属性 具体含义可查看官方文档
        Long flushInterval = context.getLongAttribute("flushInterval");
        Integer size = context.getIntAttribute("size");
        boolean readWrite = !context.getBooleanAttribute("readOnly", false);
        boolean blocking = context.getBooleanAttribute("blocking", false);
        Properties props = context.getChildrenAsProperties();
        //记录当前使用的cache对象
        builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
}
```

`MapperBuilderAssistant.useNewCache()`方法实现如下

```java
public Cache useNewCache(Class<? extends Cache> typeClass,
                         Class<? extends Cache> evictionClass,
                         Long flushInterval,
                         Integer size,
                         boolean readWrite,
                         boolean blocking,
                         Properties props) {
    //创建CacheBuilder对象来构建cache
    Cache cache = new CacheBuilder(currentNamespace)
        //配置实现类，默认为PerpetualCache
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        //添加装饰器
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        //设置属性
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    //将cache对象加入到configuration中
    configuration.addCache(cache);
    //记录当前使用的cache
    currentCache = cache;
    return cache;
}
```

`CacheBuilder`是`Cache`的构建类，属性如下

```java
//id
private String id;
//cache接口的实现类
private Class<? extends Cache> implementation;
//装饰器集合
private List<Class<? extends Cache>> decorators;
//缓存大小
private Integer size;
//清理时间周期
private Long clearInterval;
//是否可读写
private boolean readWrite;
//其他属性
private Properties properties;
//是否阻塞
private boolean blocking;
```

`CacheBuilder.build()`用来构建`cache`对象，

```java
public Cache build() {
    //设置默认实现类，该方法会设置implementation属性为PerpetualCache
    //并将LruCache加到decorators集合中
    setDefaultImplementations();
    //使用implementation属性创建实现类
    Cache cache = newBaseCacheInstance(implementation, id);
    //设置属性
    setCacheProperties(cache);
    if (PerpetualCache.class.equals(cache.getClass())) {
        //遍历decorators集合，设置装饰器
        for (Class<? extends Cache> decorator : decorators) {
            cache = newCacheDecoratorInstance(decorator, cache);
            //设置属性
            setCacheProperties(cache);
        }
        //设置一些默认的装饰器
        cache = setStandardDecorators(cache);
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
        //加入日志记录装饰器
        cache = new LoggingCache(cache);
    }
    return cache;
}
```

`setStandardDecorators()`方法会根据属性来设置`cache`的装饰器

```java
private Cache setStandardDecorators(Cache cache) {
    //该方法实现比较简单，这里就不再过多介绍了
    try {
        MetaObject metaCache = SystemMetaObject.forObject(cache);
        if (size != null && metaCache.hasSetter("size")) {
            metaCache.setValue("size", size);
        }
        if (clearInterval != null) {
            cache = new ScheduledCache(cache);
            ((ScheduledCache) cache).setClearInterval(clearInterval);
        }
        if (readWrite) {
            cache = new SerializedCache(cache);
        }
        cache = new LoggingCache(cache);
        cache = new SynchronizedCache(cache);
        if (blocking) {
            cache = new BlockingCache(cache);
        }
        return cache;
    } catch (Exception e) {
        throw new CacheException("...");
    }
}
```



##### 解析&lt;resultMap&gt;节点

介绍此方法前，首先介绍此处使用的数据结构，以下面的节点举例

```xml
<resultMap id="selectAuthor" type="org.apache.ibatis.domain.blog.Author">
    <id column="id" property="id" />
    <result property="username" column="username" />
    <result property="password" column="password" />
    <result property="email" column="email" />
    <result property="bio" column="bio" />
    <result property="favouriteSection" column="favourite_section" />
</resultMap>
```

其中，每个`resultMap`节点都会被解析`ResultMap`对象，该对象主要字段如下

```java
//ID
private String id;
//该节点的类型
private Class<?> type;
//记录除discriminator节点之外的映射关系
private List<ResultMapping> resultMappings;
//记录ID节点
private List<ResultMapping> idResultMappings;
//记录contructor节点
private List<ResultMapping> constructorResultMappings;
//记录其它property节点
private List<ResultMapping> propertyResultMappings;
//记录此resultMao涉及到的columns集合
private Set<String> mappedColumns;
//记录discriminator节点
private Discriminator discriminator;
//是否含有resultMap节点
private boolean hasNestedResultMaps;
//是否含有select节点
private boolean hasNestedQueries;
//是否开启自动映射
private Boolean autoMapping;
```

其中，每一个`result`节点会被解析成`ResultMapping`对象，该对象主要属性如下

```java

//当前的Configuration对象
private Configuration configuration;
//节点的对应属性
private String property;
private String column;
private Class<?> javaType;
private JdbcType jdbcType;
private TypeHandler<?> typeHandler;
//对应节点的resultMap属性，通过ID引用另一个resultMap标签
private String nestedResultMapId;
//对应节点的select属性
private String nestedQueryId;
private Set<String> notNullColumns;
private String columnPrefix;
//处理后的标志，标志共两个： id 和constructor
private List<ResultFlag> flags;
//对应节点的column属性
private List<ResultMapping> composites;
private String resultSet;
private String foreignColumn;
//是否延迟加载
private boolean lazy;
```

数据结构了解之后，下面开始分析代码，

`resultMapElements()`负责解析&lt;resultMap&gt;节点，该方法会遍历所有&lt;resultMap&gt;节点，依次调用`resultMapElement()`

```java
private void resultMapElements(List<XNode> list) throws Exception {
    for (XNode resultMapNode : list) {
        try {
            resultMapElement(resultMapNode);
        } catch (IncompleteElementException e) {
            // 如果解析失败，在resultMapElement()里面进行处理，此处不处理
        }
    }
}

private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.<ResultMapping> emptyList());
}

private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings) throws Exception {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    //获取该节点的ID
    String id = resultMapNode.getStringAttribute("id",
                                                 resultMapNode.getValueBasedIdentifier());
    //获取节点的type属性，获取顺序为  type > ofType > resultType > javaType
    String type = resultMapNode.getStringAttribute("type",
                                                   resultMapNode.getStringAttribute("ofType",
                                                                                    resultMapNode.getStringAttribute("resultType",
                                                                                                                     resultMapNode.getStringAttribute("javaType"))));
    //获取extends属性
    String extend = resultMapNode.getStringAttribute("extends");
    //获取autoMapping
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    //查找extends属性配置的类
    Class<?> typeClass = resolveClass(type);
    Discriminator discriminator = null;
    List<ResultMapping> resultMappings = new ArrayList<ResultMapping>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    //根据不同的节点类型调用不同的方法进行处理
    for (XNode resultChild : resultChildren) {
        if ("constructor".equals(resultChild.getName())) {
            processConstructorElement(resultChild, typeClass, resultMappings);
        } else if ("discriminator".equals(resultChild.getName())) {
            discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
        } else {
            List<ResultFlag> flags = new ArrayList<ResultFlag>();
            if ("id".equals(resultChild.getName())) {
                flags.add(ResultFlag.ID);
            }
            resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
        }
    }
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
        //解析并添加到configuration对象中
        return resultMapResolver.resolve();
    } catch (IncompleteElementException  e) {
        //记录解析失败的节点
        configuration.addIncompleteResultMap(resultMapResolver);
        throw e;
    }
}

```

###### 解析&lt;constructor&gt;节点

通过修改对象属性的方式，可以满足大多数的数据传输对象`（Data Transfer Object, DTO）`以及绝大部分领域模型的要求。但有些情况下你想使用不可变类。 一般来说，很少改变或基本不变的包含引用或数据的表，很适合使用不可变类。 构造方法注入允许你在初始化时为类设置属性的值，而不用暴露出公有方法。`MyBatis `也支持私有属性和私有 JavaBean 属性来完成注入，但有一些人更青睐于通过构造方法进行注入。 *constructor* 元素就是为此而生的。

`processConstructorElement()`主要用来解析`constructor`节点

```java
private void processConstructorElement(XNode resultChild, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    List<XNode> argChildren = resultChild.getChildren();
    //循环解析所有子节点
    for (XNode argChild : argChildren) {
        List<ResultFlag> flags = new ArrayList<ResultFlag>();
        //加入标识
        flags.add(ResultFlag.CONSTRUCTOR);
        if ("idArg".equals(argChild.getName())) {
            flags.add(ResultFlag.ID);
        }
        //解析并添加到集合中
        resultMappings.add(buildResultMappingFromContext(argChild, resultType, flags));
    }
}
```

`buildResultMappingFromContext()`方法会获取节点的属性，并调用`MapperBuilderAssistant.buildResultMapping()`创建出`ResultMapping`对象

```java
private ResultMapping buildResultMappingFromContext(XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    //获取节点属性
    String property = context.getStringAttribute("property");
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String nestedSelect = context.getStringAttribute("select");
    //如果没有指定resultMap属性的话，则认为是匿名的嵌套映射
    //processNestedResultMappings会解析匿名的嵌套映射，此方法会在后面详细说明
    String nestedResultMap = context.getStringAttribute("resultMap",
                                                        processNestedResultMappings(context, Collections.<ResultMapping> emptyList()));
    String notNullColumn = context.getStringAttribute("notNullColumn");
    String columnPrefix = context.getStringAttribute("columnPrefix");
    String typeHandler = context.getStringAttribute("typeHandler");
    String resultSet = context.getStringAttribute("resultSet");
    String foreignColumn = context.getStringAttribute("foreignColumn");
    boolean lazy = "lazy".equals(context.getStringAttribute("fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    //解析javaType属性所指定的类
    Class<?> javaTypeClass = resolveClass(javaType);
    //获取typeHandler所指定的类
    @SuppressWarnings("unchecked")
    Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
    //javaType的枚举类型
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    return builderAssistant.buildResultMapping(resultType, property, column, javaTypeClass, jdbcTypeEnum, nestedSelect, nestedResultMap, notNullColumn, columnPrefix, typeHandlerClass, flags, resultSet, foreignColumn, lazy);
}
```

`MapperBuilderAssistant.buildResultMapping()`方法主要作用是调用建造者创建出`ResultMapping`对象并设置其属性

```java
public ResultMapping buildResultMapping(
    Class<?> resultType,
    String property,
    String column,
    Class<?> javaType,
    JdbcType jdbcType,
    String nestedSelect,
    String nestedResultMap,
    String notNullColumn,
    String columnPrefix,
    Class<? extends TypeHandler<?>> typeHandler,
    List<ResultFlag> flags,
    String resultSet,
    String foreignColumn,
    boolean lazy) {
    Class<?> javaTypeClass = resolveResultJavaType(resultType, property, javaType);
    TypeHandler<?> typeHandlerInstance = resolveTypeHandler(javaTypeClass, typeHandler);
    //解析column 属性佳，当column 是” ｛ propl=coll , prop2=col2 ｝” 形式时， 会解析成ResultMapping
    //对象集合， 主要用于嵌套查询的参数传递
    List<ResultMapping> composites = parseCompositeColumnName(column);
    if (composites.size() > 0) {
        column = null;
    }
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<ResultFlag>() : flags)
        .composites(composites)
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        .build();
}
```

###### 解析&lt;discriminator&gt;节点

有时候，一个数据库查询可能会返回多个不同的结果集（但总体上还是有一定的联系的）。 鉴别器`（discriminator）`元素就是被设计来应对这种情况的，另外也能处理其它情况，例如类的继承层次结构。 鉴别器的概念很好理解——它很像` Java `语言中的 `switch `语句。

`processDiscriminatorElement()`方法负责解析`discriminator`节点

```java
private Discriminator processDiscriminatorElement(XNode context, Class<?> resultType, List<ResultMapping> resultMappings) throws Exception {
    //获取属性
    String column = context.getStringAttribute("column");
    String javaType = context.getStringAttribute("javaType");
    String jdbcType = context.getStringAttribute("jdbcType");
    String typeHandler = context.getStringAttribute("typeHandler");
    Class<?> javaTypeClass = resolveClass(javaType);
    //解析出TypeHandler
    @SuppressWarnings("unchecked")
    Class<? extends TypeHandler<?>> typeHandlerClass = (Class<? extends TypeHandler<?>>) resolveClass(typeHandler);
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    //用来记录discriminator下配置的子节点
    Map<String, String> discriminatorMap = new HashMap<String, String>();
    //遍历子节点
    for (XNode caseChild : context.getChildren()) {
        String value = caseChild.getStringAttribute("value");
        //如果没有配置resultMap，证明是匿名嵌套
        String resultMap = caseChild.getStringAttribute("resultMap", processNestedResultMappings(caseChild, resultMappings));
        discriminatorMap.put(value, resultMap);
    }
    //构建出Discriminator对象
    return builderAssistant.buildDiscriminator(resultType, column, javaTypeClass, jdbcTypeEnum, typeHandlerClass, discriminatorMap);
}
```

`processNestedResultMappings()`方法实现如下

````java
private String processNestedResultMappings(XNode context, List<ResultMapping> resultMappings) throws Exception {
    //只解析association||collection||case节点
    if ("association".equals(context.getName())
        || "collection".equals(context.getName())
        || "case".equals(context.getName())) {
        //如果没有select属性，则解析出ResultMap对象返回
        if (context.getStringAttribute("select") == null) {
            ResultMap resultMap = resultMapElement(context, resultMappings);
            return resultMap.getId();
        }
    }
    return null;
}
````

###### 解析其它节点

其它节点（如`result、id`）等节点直接调用`buildResultMappingFromContext()`进行解析，此方法之前已经分析过，此处不再叙述。

## 结语

由于篇幅原因，解析&lt;sql&gt;节点、select|insert|update|delete等SQL节点放在下篇文章进行分析。

