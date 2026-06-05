---
title: MyBatis源码分析(20)-核心处理层之SqlNode&&SqlSource(1)
tags:
  - MyBatis
date: 2019-03-23
categories:
 - MyBatis
---

### `SqlNode&&SqlSource`

通过前面的学习我们可以得知，在`MyBatis`中，每一个`SQL`节点会被解析成一个`MappedStatement`对象。其中的`SQL`语句会被解析成`SqlSource`对象，`SQL`语句中定义的动态`SQL`标签（`if，where`标签等），文本节点等，则由`SqlNode`对象来表示。

#### `OgnlCache`

分析`SqlNode&&SqlSource`之前，我们先来了解一下`OgnlCache`类。`MyBatis`中，解析表达式主要依靠`Ognl`，但`Ognl`解析时间较长，所以`OgnlCache`主要负责缓存`Ognl`表达式，该类主要定义如下

```java

/**
   * 用来缓存ognl表达式
   * */
private static final Map<String, Object> expressionCache = new ConcurrentHashMap<String, Object>();

private OgnlCache() {

}

/**
   * 根据ognl表达式获取值
   * **/
public static Object getValue(String expression, Object root) {
    try {
        Map<Object, OgnlClassResolver> context = Ognl.createDefaultContext(root, new OgnlClassResolver());
        //获取值，主要关注parseExpression(expression)方法
        return Ognl.getValue(parseExpression(expression), context, root);
    } catch (OgnlException e) {
        throw new BuilderException("。。。“);
    }
}

/**
   * 该方法会向expressionCache中取值，如果缓存中不存在才会去解析
   * */
private static Object parseExpression(String expression) throws OgnlException {
    Object node = expressionCache.get(expression);
    if (node == null) {
        node = Ognl.parseExpression(expression);
        expressionCache.put(expression, node);
    }
    return node;
}
```



#### `SqlSource`

`SqlSource`接口定义如下

```java

/**
   * 根据传入的参数获取完整的SQL语句
   * @param parameterObject
   * @return
   */
BoundSql getBoundSql(Object parameterObject);
```

该接口有四个实现类，分别为

![](/images/oss/%E5%8D%9A%E5%AE%A2/MyBatis/sqlsourceUML%E5%9B%BE.png)

其中，`RawSqlSource`负责处理静态`SQL`语句，`DynamicSqlSource`负责处理动态`SQL`语句。两者都会将处理后的`SQL`语句封装成`StaticSqlSource`对象。

首先来介绍其中比较重要的`DynamicSqlSource`对象，分析`DynamicSqlSource`之前，首先了解一下`DynamicContext`和`SqlNode`。

#### `DynamicContext`

`DynamicContext`主要用于记录动态`SQL`解析后的结果，该类主要字段如下

```java
/**
   * 用户传入的参数
   * **/
private final ContextMap bindings;
/**
   * 存储解析后的sql语句
   * **/
private final StringBuilder sqlBuilder = new StringBuilder();
```

`ContextMap`是`DynamicContext`的内部类，继承了`HashMap`，重写`get()`方法，实现如下

```java
static class ContextMap extends HashMap<String, Object> {
    private static final long serialVersionUID = 2977601501966151582L;

    private MetaObject parameterMetaObject;
    public ContextMap(MetaObject parameterMetaObject) {
        this.parameterMetaObject = parameterMetaObject;
    }

    @Override
    public Object get(Object key) {
        String strKey = (String) key;
        if (super.containsKey(strKey)) {
            return super.get(strKey);
        }

        //如果没有此key，则从parameterMetaObject中获取
        if (parameterMetaObject != null) {
            return parameterMetaObject.getValue(strKey);
        }

        return null;
    }
}
```

`DynamicContext`的构造方法会为用户传入的参数创建`MetaObject`对象（`parameterObject`为用户运行时传入的参数），并初始化`ContextMap`对象

```java
public DynamicContext(Configuration configuration, Object parameterObject) {
    //对非map集合的参数创建MetaObject对象
    if (parameterObject != null && !(parameterObject instanceof Map)) {
        MetaObject metaObject = configuration.newMetaObject(parameterObject);
        bindings = new ContextMap(metaObject);
    } else {
        bindings = new ContextMap(null);
    }
    //将参数以 “_parameter”为key存储
    //将databaseId以“_databaseId”为key进行存储
    bindings.put(PARAMETER_OBJECT_KEY, parameterObject);
    bindings.put(DATABASE_ID_KEY, configuration.getDatabaseId());
}
```

`DynamicContext`最常用的两个方法如下

```java
public void appendSql(String sql) {
    sqlBuilder.append(sql);
    sqlBuilder.append(" ");
}

public String getSql() {
    return sqlBuilder.toString().trim();
}
```

#### `SqlNode`

了解了`DynamicContext`类后，我们来分析`SqlNode`接口

`SqlNode`的接口定义如下

```java
/**
   * 此方法用来解析sql语句，并且将解析的结果保存DynamicContext对象中
   * 返回是否处理成功
   * */
boolean apply(DynamicContext context);
```

`SqlNode`接口有以下实现类

![](/images/oss/%E5%8D%9A%E5%AE%A2/MyBatis/sqlnodeUML%E5%9B%BE.png)

由于动态`SQL`语言存在嵌套情况（如`if`标签下嵌套`foreach`标签），为了方便处理，`MyBatis`此处使用了**组合模式**，如果你对此模式还不是很熟悉，建议看一看[这篇文章](https://suiyueranzly.gitee.io/posts/1191195587/)或者去网上看一些相关资料。

下面我们依次介绍每一个实现类

##### `StaticTextSqlNode`

`StaticTextSqlNode`负责解析静态`SQL`语句，实现比较简单，不需要对`SQL`语句做任何处理

```java
@Override
public boolean apply(DynamicContext context) {
    context.appendSql(text);
    return true;
}
```

##### `MixedSqlNode`

前面提到过，`Mybatis`在此处使用了组合模式，`MixedSqlNode`主要扮演组合模式中的`Composite`--组合节点角色

`MixedSqlNode`中使用`List<SqlNode> contents`字段来存储嵌套的对象，其`apply()`方法实现比较简单，就是递归调用`contents`中的`apply()`方法

```java
//用来保存嵌套对象
private List<SqlNode> contents;

public MixedSqlNode(List<SqlNode> contents) {
    this.contents = contents;
}

@Override
public boolean apply(DynamicContext context) {
    for (SqlNode sqlNode : contents) {
        sqlNode.apply(context);
    }
    return true;
}
```

##### `TextSqlNode`

`TextSqlNode`表示的是含有"${}"符号的`SQL`语句，`TextSqlNode.isDynamic()`方法前面已经分析过，这里不再阐述。`apply()`方法会把"${}"符号替换为用户实际传入的参数

```java
      @Override
      public boolean apply(DynamicContext context) {
        //创建GenericTokenParser解析"${}"符号
        GenericTokenParser parser = createParser(new BindingTokenParser(context, injectionFilter));
        context.appendSql(parser.parse(text));
        return true;
      }
```

`createParser()`方法用来创建`GenericTokenParser`对象

```java
private GenericTokenParser createParser(TokenHandler handler) {
    //处理"${}"符号，此处的handler为BindingTokenParser
    return new GenericTokenParser("${", "}", handler);
}
```

`BindingTokenParser`继承了`TokenHandler`，是`TextSqlNode`中的内部类，负责将"${}"表达式转换为实际的值，其`handleToken()`方法如下

```java
public String handleToken(String content) {
    //获取用户传入的参数
    Object parameter = context.getBindings().get("_parameter");
    if (parameter == null) {
        context.getBindings().put("value", null);
    } else if (SimpleTypeRegistry.isSimpleType(parameter.getClass())) {
        context.getBindings().put("value", parameter);
    }
    //获取实际值
    Object value = OgnlCache.getValue(content, context.getBindings());
    String srtValue = (value == null ? "" : String.valueOf(value));
    //校验参数合法性
    checkInjection(srtValue);
    return srtValue;
}
```

##### `ForEachSqlNode`

`ForEachSqlNode`表示的是动态`SQL`语句中的`foreach`标签，该类主要字段如下

```java
/**
   * 循环中每一项的固定前缀
   * */
public static final String ITEM_PREFIX = "__frch_";

/**
   * 用来解析循环终止的判断条件
   * */
private ExpressionEvaluator evaluator;
/**
   * foreach标签中的集合表达式
   * */
private String collectionExpression;
/**
   * foreach标签中嵌套的sql语句
   * */
private SqlNode contents;
/**
   * foreach标签中对应的属性
   * */
private String open;
private String close;
private String separator;
/**
   * 集合中的index和item，如果集合是map集合
   * 则index为map中的value，item为map中的value
   * */
private String item;
private String index;
/**
   * 全局配置的Configuration
   * */
private Configuration configuration;
```

该类的`apply()`方法实现如下

```java
@Override
public boolean apply(DynamicContext context) {
    //获取用户传入的参数
    Map<String, Object> bindings = context.getBindings();
    //根据集合表达式解析出集合
    final Iterable<?> iterable = evaluator.evaluateIterable(collectionExpression, bindings);
    if (!iterable.iterator().hasNext()) {
        return true;
    }
    //标记是第一次解析
    boolean first = true;
    //拼接open属性中配置的字符串
    applyOpen(context);
    int i = 0;
    //开始遍历集合
    for (Object o : iterable) {
        DynamicContext oldContext = context;
        //根据是否是第一次来决定是否要拼接分隔符，也就是separator中配置的字符串
        if (first) {
            context = new PrefixedContext(context, "");
        } else if (separator != null) {
            context = new PrefixedContext(context, separator);
        } else {
            context = new PrefixedContext(context, "");
        }
        int uniqueNumber = context.getUniqueNumber();
        if (o instanceof Map.Entry) {
            @SuppressWarnings("unchecked") 
            Map.Entry<Object, Object> mapEntry = (Map.Entry<Object, Object>) o;
            //绑定index和item
            applyIndex(context, mapEntry.getKey(), uniqueNumber);
            applyItem(context, mapEntry.getValue(), uniqueNumber);
        } else {
            applyIndex(context, i, uniqueNumber);
            applyItem(context, o, uniqueNumber);
        }
        //使用FilteredDynamicContext类解析表达式
        contents.apply(new FilteredDynamicContext(configuration, context, index, item, uniqueNumber));
        if (first) {
            first = !((PrefixedContext) context).isPrefixApplied();
        }
        context = oldContext;
        i++;
    }
    //拼接close中配置的元素
    applyClose(context);
    return true;
}
```

下面对使用到的各个方法和每个类进行具体分析

`applyOpen()和applyClose()`两个方法比较简单，直接调用`DynamicContext.appendSql()`进行拼接

```java
private void applyOpen(DynamicContext context) {
    if (open != null) {
        context.appendSql(open);
    }
}

private void applyClose(DynamicContext context) {
    if (close != null) {
        context.appendSql(close);
    }
}
```

`PrefixedContext`类继承了`DynamicContext`类，为`DynamicContext`类的代理类，实现比较简单，主要字段如下

```java
/**
     * 原始DynamicContext对象
     * */
private DynamicContext delegate;
/**
     * 要拼接的前缀
     * */
private String prefix;
/**
     * 标记是否已经拼接过了
     * */
private boolean prefixApplied;

```

该类重写了`appendSql()`方法，此方法中会判断是否已经拼接过前缀

```java
@Override
public void appendSql(String sql) {
    //判断是否已经拼接过了
    if (!prefixApplied && sql != null && sql.trim().length() > 0) {
        delegate.appendSql(prefix);
        prefixApplied = true;
    }
    delegate.appendSql(sql);
}

```

`applyIndex()和applyItem()`方法会将`index`和`item`属性绑定到`DynamicContext.bindings`集合中

```java
private void applyIndex(DynamicContext context, Object o, int i) {
    if (index != null) {
        context.bind(index, o);
        //itemizeItem方法会将index和item拼接成固定格式
        context.bind(itemizeItem(index, i), o);
    }
}

private void applyItem(DynamicContext context, Object o, int i) {
    if (item != null) {
        context.bind(item, o);
        context.bind(itemizeItem(item, i), o);
    }
}


private static String itemizeItem(String item, int i) {
    //ITEM_PREFIX前面已经介绍过 该字段为“__frch_”
    //举个例子，如果用户传入的index为“index”，当前索引为1
    //此处会拼接成“__frch_index_1”
    return new StringBuilder(ITEM_PREFIX).append(item).append("_").append(i).toString();
}
```

`FilteredDynamicContext`继承了`DynamicContext`，该类主要方法为`appendSql()`

```java
@Override
public void appendSql(String sql) {
    //拼接之前先替换，此操作会将“#{}”表达式的内容替换为前面分析的“__frch_index_1”格式
    GenericTokenParser parser = new GenericTokenParser("#{", "}", new TokenHandler() {
        @Override
        public String handleToken(String content) {
            String newContent = content.replaceFirst("^\\s*" + item + "(?![^.,:\\s])", itemizeItem(item, index));
            if (itemIndex != null && newContent.equals(content)) {
                newContent = content.replaceFirst("^\\s*" + itemIndex + "(?![^.,:\\s])", itemizeItem(itemIndex, index));
            }
            return new StringBuilder("#{").append(newContent).append("}").toString();
        }
    });
    //拼接
    delegate.appendSql(parser.parse(sql));
}
```

##### `IfSqlNode`

`IfSqlNode`表示动态`SQL`语句中的`if`标签，主要字段如下

```java
/**
   * 用来解析表达式
   * */
private ExpressionEvaluator evaluator;
/**
   * if标签中test属性的内容
   * */
private String test;
/**
   * 嵌套的sql语句
   * */
private SqlNode contents;
```

`apply()`方法实现比较简单

```java
public boolean apply(DynamicContext context) {
    //判断是否为true
    if (evaluator.evaluateBoolean(test, context.getBindings())) {
        //执行
        contents.apply(context);
        return true;
    }
    return false;
}
```

`ExpressionEvaluator.evaluateBoolean()`方法用来校验表达式是否正确

```java
public boolean evaluateBoolean(String expression, Object parameterObject) {
    //首先根据表达式取值
    Object value = OgnlCache.getValue(expression, parameterObject);
    //如果是boolean类型，则直接返回
    if (value instanceof Boolean) {
        return (Boolean) value;
    }
    //如果是数字类型，校验是否为0
    if (value instanceof Number) {
        return !new BigDecimal(String.valueOf(value)).equals(BigDecimal.ZERO);
    }
    //判断是否为空
    return value != null;
}
```

##### `VarDeclSqlNode`

`VarDeclSqlNode`表示的是`SQL`语句中的`bind`标签，该类实现比较简单

```java
//bind标签中的name属性
private final String name;
//bind标签中的表达式
private final String expression;

public VarDeclSqlNode(String var, String exp) {
    name = var;
    expression = exp;
}

@Override
public boolean apply(DynamicContext context) {
    //首先根据表达式获取值
    final Object value = OgnlCache.getValue(expression, context.getBindings());
    //添加到DynamicContext.bindings集合中
    context.bind(name, value);
    return true;
}
```

##### `ChooseSqlNode`

`ChooseSqlNode`表示`SQL`语句中的`choose`标签，该类使用`List<SqlNode>`来保存每一个`when`标签，实现如下

```java
/**
   * 保存otherwize标签下的SQL
   * */
private SqlNode defaultSqlNode;
/**
   * 保存when标签下的SQL
   * */
private List<SqlNode> ifSqlNodes;

public ChooseSqlNode(List<SqlNode> ifSqlNodes, SqlNode defaultSqlNode) {
    this.ifSqlNodes = ifSqlNodes;
    this.defaultSqlNode = defaultSqlNode;
}

@Override
public boolean apply(DynamicContext context) {
    //依次判断每一个分支，如果符合条件则执行
    for (SqlNode sqlNode : ifSqlNodes) {
        if (sqlNode.apply(context)) {
            return true;
        }
    }
    //如果都不符合则执行otherwize
    if (defaultSqlNode != null) {
        defaultSqlNode.apply(context);
        return true;
    }
    return false;
}
```

##### `TrimSqlNode&&WhereSqlNode&&SetSqlNode`

###### `TrimSqlNode`

`TrimSqlNode`表示的是`trim`标签，主要字段如下

```java
  /***
   * trim标签下嵌套的SQL
   * */
  private SqlNode contents;
  /***
   * trim标签中配置的对应属性
   * */
  private String prefix;
  private String suffix;
  /***
   * 由于prefixesToOverride和suffixesToOverride可能存在配置多个
   * 如“1|2|3|4”的情况，所以这里使用集合存储
   * */
  private List<String> prefixesToOverride;
  private List<String> suffixesToOverride;
  /***
   * 全局配置对象
   * */
  private Configuration configuration;
```

`TrimSqlNode`的构造方法中会根据用户传入的`prefixesToOverride`和`suffixesToOverride`调用`parseOverrides()`方法来初始化`prefixesToOverride、suffixesToOverride`两个字段

```java
private static List<String> parseOverrides(String overrides) {
    if (overrides != null) {
        //根据"|"进行分割
        final StringTokenizer parser = new StringTokenizer(overrides, "|", false);
        final List<String> list = new ArrayList<String>(parser.countTokens());
        //封装为集合返回
        while (parser.hasMoreTokens()) {
             //转换为大写
            list.add(parser.nextToken().toUpperCase(Locale.ENGLISH));
        }
        return list;
    }
    return Collections.emptyList();
}
```

`TrimSqlNode.apply()`方法实现比较简单，如下

```java
@Override
public boolean apply(DynamicContext context) {
    FilteredDynamicContext filteredDynamicContext = new FilteredDynamicContext(context);
    boolean result = contents.apply(filteredDynamicContext);
    filteredDynamicContext.applyAll();
    return result;
}
```

`FilteredDynamicContext`是`TrimSqlNode`的内部类，`apply()`和`appendSql()`方法实现如下

```java
@Override
public void appendSql(String sql) {
    sqlBuffer.append(sql);
}

public void applyAll() {
    //获取sql语句
    sqlBuffer = new StringBuilder(sqlBuffer.toString().trim());
    //转换为大写
    String trimmedUppercaseSql = sqlBuffer.toString().toUpperCase(Locale.ENGLISH);
    if (trimmedUppercaseSql.length() > 0) {
        //处理前缀
        applyPrefix(sqlBuffer, trimmedUppercaseSql);
        //处理后缀
        applySuffix(sqlBuffer, trimmedUppercaseSql);
    }
    //拼接sql
    delegate.appendSql(sqlBuffer.toString());
}
```

`applyPrefix()、applySuffix()`实现如下

```java
private void applyPrefix(StringBuilder sql, String trimmedUppercaseSql) {
    //如果还没有处理过
    if (!prefixApplied) {
        //标记已经处理
        prefixApplied = true;
        //首先去除遍历prefixesToOverride集合
        //删除prefixesToOverride中指定的值
        if (prefixesToOverride != null) {
            for (String toRemove : prefixesToOverride) {
                if (trimmedUppercaseSql.startsWith(toRemove)) {
                    sql.delete(0, toRemove.trim().length());
                    break;
                }
            }
        }
        //拼接前缀
        if (prefix != null) {
            sql.insert(0, " ");
            sql.insert(0, prefix);
        }
    }
}

private void applySuffix(StringBuilder sql, String trimmedUppercaseSql) {
    //如果还没有处理过
    if (!suffixApplied) {
        //标记已经处理
        suffixApplied = true;
        //首先去除遍历suffixesToOverride集合
        //删除suffixesToOverride中指定的值
        if (suffixesToOverride != null) {
            for (String toRemove : suffixesToOverride) {
                if (trimmedUppercaseSql.endsWith(toRemove) || trimmedUppercaseSql.endsWith(toRemove.trim())) {
                    int start = sql.length() - toRemove.trim().length();
                    int end = sql.length();
                    sql.delete(start, end);
                    break;
                }
            }
        }
        //添加后缀
        if (suffix != null) {
            sql.append(" ");
            sql.append(suffix);
        }
    }
}
```

###### `WhereSqlNode`

`WhereSqlNode`表示`SQL`语句中的`where`标签，该类实现比较简单，继承了`TrimSqlNode`，并将`prefix`设为`WHERE`，`prefixesToOverride`设为`"AND ","OR ","AND\n", "OR\n", "AND\r", "OR\r", "AND\t", "OR\t"`其它两个字段为空

###### `SetSqlNode`

`SetSqlNode`表示`SQL`语句中的`set`标签，同样继承`TrimSqlNode`，并将`prefix`设为`SET`，`suffixesToOverride`设置为`","`



