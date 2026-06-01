---
title: MyBatis源码分析(7)-基础支持层之类型转换
tags:
  - MyBatis
date: 2018-09-03
categories:
 - MyBatis
---



## 类型转换

`JDBC`中的类型和`Java`语言中的数据类型并不是对应的，所以`JDBC`在为`Sql`语句绑定参数时，需要将`Java`类型转换为`JDBC`类型。同理，在接收结果集时，需要将`JDBC`类型转换为`Java`类型。`MyBatis`使用类型转换器来完成上述两种操作。需要了解的是，`MyBatis`中使用`JdbcType`枚举类来存储`JDBC`类型，其中，该类中`TYPE_CODE`字段封装了`java.sql.Types`中定义的`SQL type code`，并且通过`Map<Integer,JdbcType> codeLookup`存储`SQL type code`和`JdbcType`枚举的对应关系，代码比较简单，这里就不贴出了。



### TypeHandler

`MyBatis`中所有类型处理器都实现了`TypeHandler`，`TypeHandler`中包含四个方法，分为两类：`setParameter()`将`Java`语言数据类型转换为`JDBC`类型，`getResult()`方法负责从结果集中将`JDBC`类型转换为`Java`类型，下面一起看下代码。

```java
//将Java数据类型赋值到PreparedStatement中
void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

//从结果集ResultSet中获取数据
T getResult(ResultSet rs, String columnName) throws SQLException;

T getResult(ResultSet rs, int columnIndex) throws SQLException;

T getResult(CallableStatement cs, int columnIndex) throws SQLException;
```

### BaseTypeHandler

为了方便用户自己实现此接口，`MyBatis`中提供了默认的实现类`BaseTypeHandler`，此类是一个抽象类，类关系图如下：

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/BaseTypeHandler%E7%B1%BB%E5%9B%BE.png)



此类中实现了`TypeHandler.setParameter()`和`TypeHandler.getResult()`，不过对于非空数据的处理都交给了子类实现。代码如下

```java
@Override
public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
    if (parameter == null) {
        if (jdbcType == null) {
            throw new TypeException("...");
        }
        try {
            //这里只实现了对空值的处理
            ps.setNull(i, jdbcType.TYPE_CODE);
        } catch (SQLException e) {
            throw new TypeException("... ");
        }
    } else {
        try {
            //setNonNullParameter是一个抽象方法，需要子类自行实现
            setNonNullParameter(ps, i, parameter, jdbcType);
        } catch (Exception e) {
            throw new TypeException("...");
        }
    }
}

@Override
public T getResult(ResultSet rs, String columnName) throws SQLException {
    T result;
    try {
        //getNullableResult是一个抽象方法，需要子类自行实现
        result = getNullableResult(rs, columnName);
    } catch (Exception e) {
        throw new ResultMapException("...");
    }
    if (rs.wasNull()) {	
        return null;
    } else {
        return result;
    }
}

//其它getResult方法重载实现逻辑类似
```

`BaseTypeHandler`类的子类比较多，如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/BaseTypeHandler%E5%AD%90%E7%B1%BB1.png)

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/BaseTypeHandler%E5%AD%90%E7%B1%BB2.png)

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/BaseTypeHandler%E5%AD%90%E7%B1%BB3.png)

可能有的同学看到这么多子类有点慌，不要着急，这么多类，其实实现代码逻辑都是相同的，基本都是调用`PreparedStatement.get()/PreparedStatement.set()`两个方法来赋值和获取结果集。这里我们只看一下`IntegerTypeHandler`

```java
@Override
public void setNonNullParameter(PreparedStatement ps, int i, Integer parameter, JdbcType jdbcType)
    throws SQLException {
    ps.setInt(i, parameter);
}

@Override
public Integer getNullableResult(ResultSet rs, String columnName)
    throws SQLException {
    return rs.getInt(columnName);
}

//其它重载方法省略
```

需要注意的是，`TypeHandler`一般用于完成单个值或者单个参数的获取和绑定。如果存在多个值转换为一个对象的情况，应该优先考虑使用`<resultMap>`标签来完成映射配置。

### TypeHandlerRegistry

上面介绍了一堆`TypeHandler`的实现类，那`MyBatis`是如何管理这么多实现类的呢？如何决定什么时候使用哪个实现类呢，这个伟大的任务就交给我们将要介绍的这位`TypeHandlerRegistry`。在`MyBatis`初始化过程中，会为所有已经的`TypeHandler`创建对象，并且注册到`TypeHandlerRegistry`中，由`TypeHandlerRegistry`进行统一管理，下面先看一下此类的几个字段。

```java
//保存了JdbcType和TypeHandler的映射关系
private final Map<JdbcType, TypeHandler<?>> JDBC_TYPE_HANDLER_MAP = new EnumMap<JdbcType, TypeHandler<?>>(JdbcType.class);

//保存了Java数据类型和TypeHandler的映射关系
//举例：由于Java中的String可能转换成为varchar,nvarchar等多个类型，所以这里为一对多的关系
private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new ConcurrentHashMap<Type, Map<JdbcType, TypeHandler<?>>>();

//未知类型的TypeHandler处理器
private final TypeHandler<Object> UNKNOWN_TYPE_HANDLER = new UnknownTypeHandler(this);

//所有TypeHandler的集合
private final Map<Class<?>, TypeHandler<?>> ALL_TYPE_HANDLERS_MAP = new HashMap<Class<?>, TypeHandler<?>>();

//空TypeHandler集合
private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = new HashMap<JdbcType, TypeHandler<?>>();
```

#### 注册TypeHandler

`TypeHandlerRegistry.register()`方法实现了注册`TypeHandler`对象的功能，该方法会向上述的几个集合中添加`TypeHandler`对象，该方法有多个重载，调用关系如下图所示（此图来自<<MyBatis技术内幕>>），下面来分析①～⑥这六个`register()`方法，其余的`register()`方法重载主要完成强制类型转换或初始化`TypeHandler `的功能，然后调用重载①～⑥实现注册功能，故不再做详细分析。

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/TypeHandlerRegistry%E6%B3%A8%E5%86%8C%E6%96%B9%E6%B3%95%E9%87%8D%E8%BD%BD.png)

从图中可以看出，所有的方法最后都会调用重载4完成注册功能。首先我们来介绍此方法，这个方法有三个参数，分别是`Type（Java类型）, JdbcType（Jdbc中的类型）, TypeHandler<?> （TypeHandler对象）`，下面一起看一下方法的实现

```java
private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    //如果java类型不为空
    if (javaType != null) {
        //首先获取该java类型有没有被注册过
        Map<JdbcType, TypeHandler<?>> map = TYPE_HANDLER_MAP.get(javaType);
        if (map == null) {
            //如果没有则注册该类型
            map = new HashMap<JdbcType, TypeHandler<?>>();
            //将java类型和Map<JdbcType, TypeHandler<?>>的映射关系存入集合中（注意这里是一对多，原因已经讲过）
            TYPE_HANDLER_MAP.put(javaType, map);
        }
        //将处理该java类型的handler加入“一对多”中的“多”
        map.put(jdbcType, handler);
    }
    //无论是否为空都加入所有类型的集合中
    ALL_TYPE_HANDLERS_MAP.put(handler.getClass(), handler);
}
```

在①～③这个三个`register ()`方法重载中会尝试读取`TypeHandler` 类中定义的`＠MappedTypes`
注解和`＠ MappedJdbcTypes `注解，`＠MappedTypes `注解用于指明该`TypeHandler` 实现类能够处理
的`Java `类型的集合，`＠MappedJdbcTypes `注解用于指明该`TypeHandler `实现类能够处理的`JDBC`
类型集合。`register()`方法的重载①～③ 的具体实现如下：

```java
//register方法重载1
public void register(Class<?> typeHandlerClass) {
    boolean mappedTypeFound = false;
    //扫描MappedTypes注解
    MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
        //如果找到了则取出注解中规定的Java类型
        for (Class<?> javaTypeClass : mappedTypes.value()) {
            //调用重载3处理
            register(javaTypeClass, typeHandlerClass);
            mappedTypeFound = true;
        }
    }
    if (!mappedTypeFound) {
        //如果没有找到则调用重载2继续处理
        register(getInstance(null, typeHandlerClass));
    }
}


//register方法重载2
@SuppressWarnings("unchecked")
public <T> void register(TypeHandler<T> typeHandler) {
    boolean mappedTypeFound = false;
    //扫描MappedTypes注解
    MappedTypes mappedTypes = typeHandler.getClass().getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
        ////如果找到了则取出注解中规定的Java类型
        for (Class<?> handledType : mappedTypes.value()) {
            //调用重载3处理
            register(handledType, typeHandler);
            mappedTypeFound = true;
        }
    }
    // 从3.1.0开始 就可以根据TypeHandler查找对应的Java类型，前提是必须要实现TypeReference接口
    if (!mappedTypeFound && typeHandler instanceof TypeReference) {
        //如果实现了TypeReference接口
        try {
            TypeReference<T> typeReference = (TypeReference<T>) typeHandler;
            //则取出该TypeHandler类对应的Java类型，调用重载3处理
            register(typeReference.getRawType(), typeHandler);
            mappedTypeFound = true;
        } catch (Throwable t) {

        }
    }

    if (!mappedTypeFound) {
        //如果没有找到，则交给重载3进行处理
        register((Class<T>) null, typeHandler);
    }
}


//register方法重载3
private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
    //扫描MappedJdbcTypes注解
    MappedJdbcTypes mappedJdbcTypes = typeHandler.getClass().getAnnotation(MappedJdbcTypes.class);
    if (mappedJdbcTypes != null) {
        //如果找到了则取出该TypeHandler能处理的JDBC类型，调用重载4处理
        for (JdbcType handledJdbcType : mappedJdbcTypes.value()) {
            register(javaType, handledJdbcType, typeHandler);
        }
        if (mappedJdbcTypes.includeNullJdbcType()) {
            register(javaType, null, typeHandler);
        }
    } else {
        register(javaType, null, typeHandler);
    }
}
```

不知道大家有没有注意到，上面介绍的几个方法，都是在向`TYPE_HANDLER_MAP`集合中添加数据，此类中只有重载5才是向`JDBC_TYPE_HANDLER_MAP`添加数据

```java
public void register(JdbcType jdbcType, TypeHandler<?> handler) {
    JDBC_TYPE_HANDLER_MAP.put(jdbcType, handler);
}
```

而此方法会在初始化时被调用

```java
public TypeHandlerRegistry() {
    register(Boolean.class, new BooleanTypeHandler());
    register(boolean.class, new BooleanTypeHandler());
    register(JdbcType.BOOLEAN, new BooleanTypeHandler());
    register(JdbcType.BIT, new BooleanTypeHandler());
    
    //其余省略。。。
}
```

`TypeHandlerRegistry`提供了单独的`TypeHandler`对象注册，还可以通过包名批量扫描注册，下面一起来看重载6

#### 查找TypeHandler

介绍完了`TypeHandlerRegistry`的注册功能后，下面一起来看看如何查找对应的`TypeHandler`对象。`TypeHandlerRegistry.getTypeHandler()`方法提供了查找的功能，该方法也有多个重载，如下（此图来自<<MyBatis技术内幕>>）

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/TypeHandlerRegistry%E6%9F%A5%E6%89%BE%E6%96%B9%E6%B3%95%E9%87%8D%E8%BD%BD.png)



可以看到，经过一系列的调用，最终会落在`TypeHandlerRegistry.getTypeHandler(Type type, JdbcType jdbcType)`方法进行处理，参数的含义不用多说，下面来看看此方法的实现。

```java
@SuppressWarnings("unchecked")
private <T> TypeHandler<T> getTypeHandler(Type type, JdbcType jdbcType) {
    //getJdbcHandlerMap方法会根据java类型获取“一对多”中“多”的TypeHandler集合
    //后面会介绍此方法的实现
    Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = getJdbcHandlerMap(type);
    TypeHandler<?> handler = null;
    if (jdbcHandlerMap != null) {
        //如果获取到了则根据JdbcType获取相应的TypeHandler
        handler = jdbcHandlerMap.get(jdbcType);
        if (handler == null) {
            handler = jdbcHandlerMap.get(null);
        }
        if (handler == null) {
            // 如果没有获取到，则调用pickSoleHandler选出最合适的一个
            handler = pickSoleHandler(jdbcHandlerMap);
        }
    }

    return (TypeHandler<T>) handler;
}


@SuppressWarnings({ "rawtypes", "unchecked" })
private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMap(Type type) {
    //getJdbcHandlerMap方法会根据java类型获取“一对多”中“多”的TypeHandler集合
    Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = TYPE_HANDLER_MAP.get(type);
    //如果是空集合则直接返回空
    if (NULL_TYPE_HANDLER_MAP.equals(jdbcHandlerMap)) {
        return null;
    }
    //如果是null则扫描父类的TypeHandler集合，并且做为此类的集合
    if (jdbcHandlerMap == null && type instanceof Class) {
        Class<?> clazz = (Class<?>) type;
        //扫描父类集合
        //后面会介绍此方法的实现
        jdbcHandlerMap = getJdbcHandlerMapForSuperclass(clazz);
        if (jdbcHandlerMap != null) {
            //如果不等于空则做为此类的集合
            TYPE_HANDLER_MAP.put(type, jdbcHandlerMap);
        } else if (clazz.isEnum()) {
            //如果是枚举类则使用EnumTypeHandler注册
            register(clazz, new EnumTypeHandler(clazz));
            //返回结果
            return TYPE_HANDLER_MAP.get(clazz);
        }
    }
    //如果经过了上述的操作还是没有找到，则直接注册一个空的集合
    if (jdbcHandlerMap == null) {
        TYPE_HANDLER_MAP.put(type, NULL_TYPE_HANDLER_MAP);
    }
    //返回
    return jdbcHandlerMap;
}

private Map<JdbcType, TypeHandler<?>> getJdbcHandlerMapForSuperclass(Class<?> clazz) {
    //获取父类
    Class<?> superclass =  clazz.getSuperclass();
    //如果是空或者是Object（没有父类），则直接返回null
    if (superclass == null || Object.class.equals(superclass)) {
        return null;
    }
    //查找父类对应的集合
    Map<JdbcType, TypeHandler<?>> jdbcHandlerMap = TYPE_HANDLER_MAP.get(superclass);
    if (jdbcHandlerMap != null) {
        //如果不等于空则直接返回
        return jdbcHandlerMap;
    } else {
        //递归查找父类
        return getJdbcHandlerMapForSuperclass(superclass);
    }
}
```

介绍完查找方法后，一起看看其它的两个方法，`TypeHandlerRegistry.getMappingTypeHandler()`方法会根据传入的`TypeHandler`类型 向`ALL_TYPE_HANDLERS_MAP`中查找指定的`TypeHandler`对象。`TypeHandlerRegistry。getTypeHandler(JdbcType)`方法会根据指定的`JdbcType`向`JDBC_TYPE_HANDLER_MAP`集合中获取`TypeHandler`对象。

除了`MyBatis`中定义的`TypeHandler`实现类，我们也可以自定义实现类，添加方式是在`mybatis-config.xml`配置文件中的`<typeHandlers>`节点下， 添加相应的`<typeHandler>`节点配置，并指定自定义的`TypeHandler` 接口实现类，在`MyBatis`初始化时，会将此实现类注册到`TypeHandlerRegistry`中，这里以后会提到。







