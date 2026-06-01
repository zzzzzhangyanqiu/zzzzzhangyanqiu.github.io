---
title: MyBatis源码分析(21)-核心处理层之SqlNode&&SqlSource(2)
tags:
  - MyBatis
date: 2019-03-30
categories:
 - MyBatis
---

##### `SqlSourceBuilder`

经过`SqlNode`解析后，`SQL`语句会被传递到`SqlSourceBuilder`进行进一步的解析。`SqlSourceBuilder`的解析主要分为两方面，一方面是对`SQL`语句中的`"#{}"`占位符进行解析，替换成`"?"`。一方面是对`"#{}"`占位符中的自定义格式进行解析，这里的自定义格式类似于前面提到过的`foreach`标签解析出来的格式，如`#{__frch_index_1}、#{__frch_item_0, javaType= int, jdbcType=NUMERIC, typeHandler=MyTypeHandler}`等。

`SqlSourceBuilder`继承了`BaseBuilder(该类之前已经分析过)`，其核心方法是`parse()`，实现如下

```java
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
    //创建ParameterMappingTokenHandler来解析"#{}"符号
    ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    //开始解析sql
    String sql = parser.parse(originalSql);
    //将解析后的sql语句封装成StaticSqlSource返回
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
}
```

`ParameterMappingTokenHandler`是该类的一个内部类，实现了`TokenHandler`，主要字段如下

```java
    /**
     * 记录解析表达式后得到的结果
     * **/
    private List<ParameterMapping> parameterMappings = new ArrayList<ParameterMapping>();

    /**
     * 参数类型
     * **/
    private Class<?> parameterType;

    /**
     * DynamicContext.bindings集合对应的MetaObject对象
     * **/
    private MetaObject metaParameters;
```

表达式解析后会被转换为`ParameterMapping`类，`ParameterMapping`类的主要字段如下

```java
  /**
   * 全局配置对象
   * **/
  private Configuration configuration;

  /**
   * property属性
   * */
  private String property;

  /**
   * 输入参数还是输出参数
   * */
  private ParameterMode mode;

  /**
   * 参数的javaType
   * **/
  private Class<?> javaType = Object.class;

  /**
   * 参数的JdbcType
   * **/
  private JdbcType jdbcType;

  /**
   * 浮点参数的精度
   * **/
  private Integer numericScale;

  /**
   * 参数对应的TypeHandler
   * */
  private TypeHandler<?> typeHandler;

  /**
   * 参数对应的resultMapId
   * **/
  private String resultMapId;

  /**
   * 参数的jdbcTypeName
   * **/
  private String jdbcTypeName;
```



其`handleToken()`方法如下

```java
@Override
public String handleToken(String content) {
    //重点关注buildParameterMapping()方法
    parameterMappings.add(buildParameterMapping(content));
    //将"#{}"表达式的内容替换为"?"返回
    return "?";
}

private ParameterMapping buildParameterMapping(String content) {
    //将传入的表达式解析为map集合
    //例如#{__frch_item_0, javaType=int, jdbcType=NUMERIC, typeHandler=MyTypeHandler}这个占位符
    // 它就会被解析成如下Map
    //property -> __frch_item_0, javaType -> int , jdbcType -> NUMERIC, typeHandler -> MyTypeHandler
    Map<String, String> propertiesMap = parseParameterMapping(content);
    String property = propertiesMap.get("property");
    Class<?> propertyType;
    //检测参数的jdbcType
    if (metaParameters.hasGetter(property)) {
        propertyType = metaParameters.getGetterType(property);
    } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
    } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
    } else if (property != null) {
        MetaClass metaClass = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        if (metaClass.hasGetter(property)) {
            propertyType = metaClass.getGetterType(property);
        } else {
            propertyType = Object.class;
        }
    } else {
        propertyType = Object.class;
    }
    //使用property和javaType为参数创建ParameterMapping对象
    ParameterMapping.Builder builder = new ParameterMapping.Builder(configuration, property, propertyType);
    Class<?> javaType = propertyType;
    String typeHandlerAlias = null;
    //为各个属性赋值
    for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
            javaType = resolveClass(value);
            builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
            builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {
            builder.mode(resolveParameterMode(value));
        } else if ("numericScale".equals(name)) {
            builder.numericScale(Integer.valueOf(value));
        } else if ("resultMap".equals(name)) {
            builder.resultMapId(value);
        } else if ("typeHandler".equals(name)) {
            typeHandlerAlias = value;
        } else if ("jdbcTypeName".equals(name)) {
            builder.jdbcTypeName(value);
        } else if ("property".equals(name)) {
            // 由于前面已经处理过property，这里不需要处理了
        } else if ("expression".equals(name)) {
            //目前还不支持expression属性
            throw new BuilderException("....");
        } else {
            throw new BuilderException("......");
        }
    }
    if (typeHandlerAlias != null) {
        //复制typeHandler
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
    }
    return builder.build();
}
```

##### `BoundSql`

前面我们在叙述`SqlSource`接口时，讲到`getBoundSql()`方法会返回`BoundSql`对象，其主要字段和方法如下

```java
  /**
   * SQL语句
   * **/
  private String sql;

  /**
   * SQL语句中的参数集合
   * **/
  private List<ParameterMapping> parameterMappings;

  /**
   * 运行时传入的实际参数
   * **/
  private Object parameterObject;

  /**
   * 此集合其实就是DynamicContext.bindings集合
   * **/
  private Map<String, Object> additionalParameters;

  /**
   * DynamicContext.bindings集合对应的MetaObject对象
   * **/
  private MetaObject metaParameters;


  /**
   * 判断是否有此参数
   * */
  public boolean hasAdditionalParameter(String name) {
    String paramName = new PropertyTokenizer(name).getName();
    return additionalParameters.containsKey(paramName);
  }
```

##### `DynamicSqlSource`

了解前面的分析之后，对于`DynamicSqlSource`的理解就非常容易了，`DynamicSqlSource`继承了`SqlSource`，其`getBoundSql()`方法实现如下

```java
@Override
public BoundSql getBoundSql(Object parameterObject) {
    //创建DynamicContext对象
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    //解析SQL语句
    rootSqlNode.apply(context);
    //创建SqlSourceBuilder
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    //解析参数类型
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    //进一步解析SQL语句
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    //获取sql语句，由于SqlSourceBuilder.parse()方法会返回StaticSqlSource
    //稍后我们会分析StaticSqlSource.getBoundSql()
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    //为BoundSql.additionalParameters集合复制
    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
        boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
    }
    return boundSql;
}
```

`StaticSqlSource.getBoundSql()`方法实现比较简单，直接创建`BoundSql`对象返回

```java
@Override
public BoundSql getBoundSql(Object parameterObject) {
    return new BoundSql(configuration, sql, parameterMappings, parameterObject);
}
```

##### `RawSqlSource`

前面我们讲过，`RawSqlSource`主要负责解析静态`SQL`语句。`RawSqlSource`逻辑与`DynamicSqlSource`类似，前面在讲解`XMLScriptBuilder.parseScriptNode()`时提过，`MyBatis`会根据是否是动态SQL来选择创建`DynamicSqlSource`或`RawSqlSource`

`RawSqlSource`的构造方法会调用自身的`getSql`获取`SQL`语句，由于使用到了组合模式，该方法实现较为简单

```java
public RawSqlSource(Configuration configuration, SqlNode rootSqlNode, Class<?> parameterType) {
    //调用自身的重载构造方法
    this(configuration, getSql(configuration, rootSqlNode), parameterType);
}

public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
    //创建SqlSourceBuilder对象并解析sql
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> clazz = parameterType == null ? Object.class : parameterType;
    sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<String, Object>());
}

private static String getSql(Configuration configuration, SqlNode rootSqlNode) {
    DynamicContext context = new DynamicContext(configuration, null);
    //组合模式，只需处理一个节点
    rootSqlNode.apply(context);
    return context.getSql();
}
```

`RawSqlSource.getBoundSql()`直接调用`StaticSqlSource.getBoundSql()`方法

```java
@Override
public BoundSql getBoundSql(Object parameterObject) {
    return sqlSource.getBoundSql(parameterObject);
}
```

##### 总结

通过这两篇文章的分析，我们知道`StaticSqlSource、RawSqlSource、DynamicSqlSource`都会将`SQL`节点解析成`BoundSql`返回，`BoundSql`中封装了已经把表达式解析成`"?"`的`SQL`语句，参数映射的关系，用户传入的参数集合等。

`RawSqlSource`和`StaticSqlSource`实际看起来逻辑比较类似，不过两个类的执行时机完全不一样，`StaticSqlSource`是在调用`getBoundSql()`方法时才开始解析，而`RawSqlSource`则是在`MyBatis`初始化时采取解析，大家可以思考一下，为什么要这样做？