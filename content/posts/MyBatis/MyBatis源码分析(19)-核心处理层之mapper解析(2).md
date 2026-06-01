---
title: MyBatis源码分析(19)-核心处理层之mapper解析(2)
tags:
  - MyBatis
date: 2019-02-23
categories:
 - MyBatis
---

### XMLStatementBuilder

上篇文章中由于篇幅原因，还剩下一部分内容，这里继续进行分析，`MyBatis`中，具体的`sql`语句是由`XMLStatementBuilder`负责解析。

##### 解析&lt;sql&gt;节点

这个元素可以被用来定义可重用的 SQL 代码段，这些 SQL 代码可以被包含在其他语句中。它可以（在加载的时候）被静态地设置参数。 在不同的包含语句中可以设置不同的值到参数占位符上。

`sqlElement()`方法负责解析&lt;sql&gt;节点，该方法获取`databaseId`并调用重载

```java
private void sqlElement(List<XNode> list) throws Exception {
    if (configuration.getDatabaseId() != null) {
        sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
}

private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
    for (XNode context : list) {
        String databaseId = context.getStringAttribute("databaseId");
        String id = context.getStringAttribute("id");
        //根据命名空间获取ID
        id = builderAssistant.applyCurrentNamespace(id, false);
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
            //添加到sqlFragments中，
            // 由XMLMapperBuilder的构造函数可知，该集合指向configuration对象中的sqlFragments集合
            sqlFragments.put(id, context);
        }
    }
}
```

可以看到，这里只是对`sql`节点进行记录，等到使用的时候再进行解析。大家可以思考一下，为什么要这样设计呢？

##### 解析select|insert|update|delete等SQL节点

介绍此处时，首先了解一下此处使用到的数据结构，所有的`sql`语句都会被解析成`SqlSource`对象，`SqlSource`是个接口，只有一个方法，如下

```java
/**
   * 根据传入的参数获取完整的SQL语句
   * @param parameterObject
   * @return
   */
BoundSql getBoundSql(Object parameterObject);
```

这里只需要稍微了解就可以了，以后会详细分析

`select|insert|update|delete`等SQL节点都会被解析成`MappedStatement`对象，简单介绍一下该类的主要字段

```java
private String resource;  //节点中的ID属性（包括表前缀）
private Configuration configuration;
private String id;
private Integer fetchSize;
private Integer timeout;
private StatementType statementType;
private ResultSetType resultSetType;
private SqlSource sqlSource;    //sqlSource对象，代表一条sql语句
private Cache cache;
private ParameterMap parameterMap;
private List<ResultMap> resultMaps;
private boolean flushCacheRequired;
private boolean useCache;
private boolean resultOrdered;
private SqlCommandType sqlCommandType;   //sql类型 UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH;
private KeyGenerator keyGenerator;
private String[] keyProperties;
private String[] keyColumns;
private boolean hasNestedResultMaps;
private String databaseId;
private Log statementLog;
private LanguageDriver lang;
private String[] resultSets;
```

`buildStatementFromContext()`方法负责解析`select|insert|update|delete`等`SQL`节点

该方法同上面一样，会获取`databaseId`并调用重载

```java
private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    //遍历解析
    for (XNode context : list) {
        //开始解析
        final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
        try {
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            //如果解析失败则加入到incompleteStatement集合中
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```

`XMLStatementBuilder.parseStatementNode()`负责解析具体的`SQL`节点

```java
public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
        return;
    }
    //获取节点的相关属性
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultType = context.getStringAttribute("resultType");
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    Class<?> resultTypeClass = resolveClass(resultType);
    String resultSetType = context.getStringAttribute("resultSetType");
    StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

    String nodeName = context.getNode().getNodeName();
    //根据节点属性获取sql类型
    SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    // 解析之前先处理<include>节点
    XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    includeParser.applyIncludes(context.getNode());

    // 解析之前先处理<selectKey>节点
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // 开始解析sql
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    if (configuration.hasKeyGenerator(keyStatementId)) {
        keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
        keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                                                   configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
            ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
    }

    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
                                        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
                                        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
                                        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

###### 解析&lt;include&gt;节点

`XMLIncludeTransformer.applyIncludes()`用来将`include`节点解析为可执行的`SQL`语句

```java
public void applyIncludes(Node source) {
    //存储变量集合
    Properties variablesContext = new Properties();
    //获取配置文件中定义的变量集合
    Properties configurationVariables = configuration.getVariables();
    if (configurationVariables != null) {
        variablesContext.putAll(configurationVariables);
    }
    //调用重载
    applyIncludes(source, variablesContext, false);
}
```

```java
private void applyIncludes(Node source, final Properties variablesContext, boolean included) {

    //2、如果是include节点
    if (source.getNodeName().equals("include")) {
        //获取refid中配置的sql片段ID
        Node toInclude = findSqlFragment(getStringAttribute(source, "refid"), variablesContext);
        //获取include节点配置的变量集合，与传入的进行合并
        Properties toIncludeContext = getVariablesContext(source, variablesContext);
        //解析sql节点
        applyIncludes(toInclude, toIncludeContext, true);
        //如果不在一个文档中，则导入当前文档
        if (toInclude.getOwnerDocument() != source.getOwnerDocument()) {
            toInclude = source.getOwnerDocument().importNode(toInclude, true);
        }
        //将include节点替换为sql节点
        source.getParentNode().replaceChild(toInclude, source);
        //如果有子节点，则插入到sql节点的前面
        // （该方法会将sql节点中的sql语句插入到节点的前方）
        while (toInclude.hasChildNodes()) {
            toInclude.getParentNode().insertBefore(toInclude.getFirstChild(), toInclude);
        }
        //移除sql节点
        toInclude.getParentNode().removeChild(toInclude);
    } else if (source.getNodeType() == Node.ELEMENT_NODE) {
        //1、如果类型是节点，则获取该节点下所有子节点
        NodeList children = source.getChildNodes();
        for (int i = 0; i < children.getLength(); i++) {
            //递归调用此方法
            applyIncludes(children.item(i), variablesContext, included);
        }
    } else if (included && source.getNodeType() == Node.TEXT_NODE
               //3、如果是文本节点
               && !variablesContext.isEmpty()) {
        // 替换文本中的变量
        source.setNodeValue(PropertyParser.parse(source.getNodeValue(), variablesContext));
    }
}
```

###### 解析&lt;selectKey&gt;节点

对于不支持自动生成类型的数据库或可能不支持自动生成主键的 JDBC 驱动，MyBatis 有另外一种方法来生成主键。`selectKey `元素中的语句将会首先运行，然后插入语句会被调用。这可以提供给你一个与数据库中自动生成主键类似的行为，同时保持了` Java` 代码的简洁。

`processSelectKeyNodes()`负责解析`selectKey `元素

```java
private void processSelectKeyNodes(String id, Class<?> parameterTypeClass, LanguageDriver langDriver) {
    //获取selectKey节点集合
    List<XNode> selectKeyNodes = context.evalNodes("selectKey");
    if (configuration.getDatabaseId() != null) {
        //开始解析
        parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, configuration.getDatabaseId());
    }
    parseSelectKeyNodes(id, selectKeyNodes, parameterTypeClass, langDriver, null);
    //移除掉selectKey节点
    removeSelectKeyNodes(selectKeyNodes);
}

private void parseSelectKeyNodes(String parentId, List<XNode> list, Class<?> parameterTypeClass, LanguageDriver langDriver, String skRequiredDatabaseId) {
    for (XNode nodeToHandle : list) {
        //根据规则拼接ID
        String id = parentId + SelectKeyGenerator.SELECT_KEY_SUFFIX;
        String databaseId = nodeToHandle.getStringAttribute("databaseId");
        if (databaseIdMatchesCurrent(id, databaseId, skRequiredDatabaseId)) {
            //调用重载
            parseSelectKeyNode(id, nodeToHandle, parameterTypeClass, langDriver, databaseId);
        }
    }
}

private void parseSelectKeyNode(String id, XNode nodeToHandle, Class<?> parameterTypeClass, LanguageDriver langDriver, String databaseId) {
    //获取节点的相关属性
    String resultType = nodeToHandle.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    StatementType statementType = StatementType.valueOf(nodeToHandle.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    String keyProperty = nodeToHandle.getStringAttribute("keyProperty");
    String keyColumn = nodeToHandle.getStringAttribute("keyColumn");
    boolean executeBefore = "BEFORE".equals(nodeToHandle.getStringAttribute("order", "AFTER"));

    //默认值
    boolean useCache = false;
    boolean resultOrdered = false;
    KeyGenerator keyGenerator = new NoKeyGenerator();
    Integer fetchSize = null;
    Integer timeout = null;
    boolean flushCache = false;
    String parameterMap = null;
    String resultMap = null;
    ResultSetType resultSetTypeEnum = null;

    //获取sqlsource对象
    SqlSource sqlSource = langDriver.createSqlSource(configuration, nodeToHandle, parameterTypeClass);
    //selectkey节点中只能是select类型的sql
    SqlCommandType sqlCommandType = SqlCommandType.SELECT;

    //添加到configuration.mappedStatements集合中
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
                                        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
                                        resultSetTypeEnum, flushCache, useCache, resultOrdered,
                                        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, null);

    id = builderAssistant.applyCurrentNamespace(id, false);

    MappedStatement keyStatement = configuration.getMappedStatement(id, false);
    //创建selectkey节点对应的KeyGenerator对象，并保存到configuration中
    configuration.addKeyGenerator(id, new SelectKeyGenerator(keyStatement, executeBefore));
}
```

`LanguageDriver`主要有两个接口，分别是`XMLLanguageDriver`和`RawLanguageDriver`，`configuration`的构造方法中，可以看到如下代码，我们还可以自定义实现，并且在`xml`文件中配置

```java
languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
```

`XMLLanguageDriver.createSqlSource()`方法最终会调用`XMLScriptBuilder.parseScriptNode()`方法，方法实现如下

```java
public SqlSource parseScriptNode() {
    //首先判断当前的节点是不是动态sql
    List<SqlNode> contents = parseDynamicTags(context);
    //将当前的SqlNode集合包装成MixedSqlNode，此类会在后面详细讲解
    MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
    SqlSource sqlSource = null;
    //根据是否是动态sql，创建不同的sqlsource实现类
    if (isDynamic) {
        sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
        sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
}
```

`parseDynamicTags()`方法中，会遍历`selectKey`下的每个节点，如果有标签节点或者是`${}`占位符，则被认为是动态`sql`

```java
List<SqlNode> parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<SqlNode>();
    NodeList children = node.getNode().getChildNodes();
    //遍历下面的每一个节点
    for (int i = 0; i < children.getLength(); i++) {
        XNode child = node.newXNode(children.item(i));
        //如果是文本节点
        if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
            String data = child.getStringBody("");
            //解析成TextSqlNode对象
            //根据是否是动态sql分别添加不同的SqlNode实现类
            TextSqlNode textSqlNode = new TextSqlNode(data);
            if (textSqlNode.isDynamic()) {
                contents.add(textSqlNode);
                //标记为动态sql语句
                isDynamic = true;
            } else {
                contents.add(new StaticTextSqlNode(data));
            }
        } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) {
            //如果是标签，那么一定是动态sql，根据不同的标签创建NodeHandler实现类
            String nodeName = child.getNode().getNodeName();
            //如果还存在其它节点，调用nodeHandlers()获取NodeHandler对象
            NodeHandler handler = nodeHandlers(nodeName);
            if (handler == null) {
                throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
            }
            handler.handleNode(child, contents);
            //标记为动态sql
            isDynamic = true;
        }
    }
    return contents;
}
```

`TextSqlNode.isDynamic()`方法会使用`DynamicCheckerTokenParser`类，并且调用`GenericTokenParser.parse()`方法解析传入的`sql`语句，该方法前面已经介绍过，此处不再分析

`DynamicCheckerTokenParser.handleToken()`方法实现如下

```java
/**
     * 这里可能有些同学已经忘记了，简单的说：
     * GenericTokenParser.parse()方法解析过程中，
     * 如果遇到了占位符，就会调用传入的TokenHandler.handleToken()方法
     */
public String handleToken(String content) {
    this.isDynamic = true;
    return null;
}
```



`nodeHandlers()`会根据不同的节点名称返回对应的实现类

```java
NodeHandler nodeHandlers(String nodeName) {
    Map<String, NodeHandler> map = new HashMap<String, NodeHandler>();
    map.put("trim", new TrimHandler());
    map.put("where", new WhereHandler());
    map.put("set", new SetHandler());
    map.put("foreach", new ForEachHandler());
    map.put("if", new IfHandler());
    map.put("choose", new ChooseHandler());
    map.put("when", new IfHandler());
    map.put("otherwise", new OtherwiseHandler());
    map.put("bind", new BindHandler());
    return map.get(nodeName);
}
```

获取到`NodeHandler`对象后，会调用`NodeHandler.handleNode()`方法进行处理，这里以`WhereHandler`举例说明

```java
public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
    //解析where节点的子节点，parseDynamicTags()方法前面已经介绍过
    List<SqlNode> contents = parseDynamicTags(nodeToHandle);
    MixedSqlNode mixedSqlNode = new MixedSqlNode(contents);
    //创建WhereSqlNode，并且添加到集合中
    WhereSqlNode where = new WhereSqlNode(configuration, mixedSqlNode);
    targetContents.add(where);
}
```

###### 解析sql文本

至此，`sql`中的`include`和`selectKey`标签可以全部解析完成，剩下的代码是解析`sql`文本的逻辑，实现如下

```java
public void parseStatementNode() {
    // 处理<include>节点.....
    
    // 处理<selectKey>节点....
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // 开始解析sql
    //获取SqlSource接口的实现类
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    //获取对应属性
    String resultSets = context.getStringAttribute("resultSets");
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    KeyGenerator keyGenerator;
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    //获取keyGenerator对象，keyGenerator接口后面会详细分析
    if (configuration.hasKeyGenerator(keyStatementId)) {
        keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
        keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                                                   configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
            ? new Jdbc3KeyGenerator() : new NoKeyGenerator();
    }

    //解析节点并添加到configuration.mappedStatements集合中，此处比较简单，不再叙述
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
                                        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
                                        resultSetTypeEnum, flushCache, useCache, resultOrdered, 
                                        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

### 绑定mapper接口

到此为止，一个`mapper`文件就已经解析完成了，回到`XMLMapperBuilder.parse()`方法，为了避免有些同学忘记，这里再看一次方法实现

```java
public void parse() {
    //是否已经加载过了此文件
    if (!configuration.isResourceLoaded(resource)) {
        //解析mapper
        configurationElement(parser.evalNode("/mapper"));
        //记录已经解析过此文件
        configuration.addLoadedResource(resource);
        //注册mapper接口
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

之前我们已经介绍过，所有的`mapper`类都会向`configuration.mapperRegistry`集合中注册，`bindMapperForNamespace()`方法会将解析后的`mapper`类添加到此集合中

```java
private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
        Class<?> boundType = null;
        try {
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
        }
        if (boundType != null) {
            if (!configuration.hasMapper(boundType)) {
                //记录已经解析过的namespace
                configuration.addLoadedResource("namespace:" + namespace);
                //该方法会调用mapperRegistry.addMapper()
                configuration.addMapper(boundType);
            }
        }
    }
}
```

之前在介绍`mapperRegistry.addMapper()`方法时，只说会向集合中添加数据，其实该方法还会调用`MapperAnnotationBuilder.parse()`方法来处理`mapper`类中的注解

```java
public void parse() {
    String resource = type.toString();
    //判断是否已经加载过此文件
    if (!configuration.isResourceLoaded(resource)) {
        //判断是否已经加载过此类对应的xml文件，如果没有加载过会重新进行加载
        loadXmlResource();
        configuration.addLoadedResource(resource);
        assistant.setCurrentNamespace(type.getName());
        //解析@cache注解
        parseCache();
        //解析@cacheRef注解
        parseCacheRef();
        Method[] methods = type.getMethods();
        for (Method method : methods) {
            try {
                //获取所有方法并解析@resultMap等注解，并创建MappedStatement对象添加到集合中
                if (!method.isBridge()) {
                    parseStatement(method);
                }
            } catch (IncompleteElementException e) {
                //如果解析失败，可能是引用了未解析的文件，
                // 加到incompleteMethod集合中稍后进行解析
                configuration.addIncompleteMethod(new MethodResolver(this, method));
            }
        }
    }
    //重新进行解析
    parsePendingMethods();
}
```



### 解析失败的节点处理

由于解析文件是有顺序的，所以在解析文件时，可能会引用未解析的文件导致失败，为了解决这种情况`MyBatis`在解析失败时，暂时加入到`incomplete*`集合中，在`XMLMapperBuilder.parse()`方法后面继续进行解析

```java
public void parse() {
    //解析文件。。

    //处理configurationElement中解析失败的ResultMap标签
    parsePendingResultMaps();
    //处理configurationElement中解析失败的CacheRefs标签
    parsePendingChacheRefs();
    //处理configurationElement中解析失败的SQL语句
    parsePendingStatements();
}
```

三个方法实现都差不多，这里只分析`parsePendingResultMaps()`

```java
private void parsePendingResultMaps() {
    Collection<ResultMapResolver> incompleteResultMaps = configuration.getIncompleteResultMaps();
    //同步锁
    synchronized (incompleteResultMaps) {
        Iterator<ResultMapResolver> iter = incompleteResultMaps.iterator();
        //遍历  依次进行解析
        while (iter.hasNext()) {
            try {
                iter.next().resolve();
                iter.remove();
            } catch (IncompleteElementException e) {
                // 如果还是解析失败，则放弃解析
            }
        }
    }
}
```



