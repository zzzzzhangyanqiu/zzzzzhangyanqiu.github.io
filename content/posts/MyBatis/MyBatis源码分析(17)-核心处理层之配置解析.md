---
title: MyBatis源码分析(17)-核心处理层之配置解析
tags:
  - MyBatis
date: 2019-02-08
categories:
 - MyBatis
---

### 前言

经过前面的16篇文章，我们学习了`MyBatis`的基础模块，这些模块为核心层的功能提供了技术支撑，我们在日常使用中，如果遇到类似的功能也可以借鉴对应的模块实现。从此篇开始，我们来学习`MyBatis`的核心处理层逻辑。

首先来看`MyBatis`的官方实例代码

```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

try (SqlSession sqlSession = sqlSessionFactory.openSession()) {
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    List users = mapper.getUsers();
    System.out.println(users.size());

} catch (Exception e) {
    e.printStackTrace();
}
```

前面两句是获取`MyBatis`的配置文件，不用多说，我们主要从`SqlSessionFactoryBuilder().build()`开始分析。

### SqlSessionFactoryBuilder().build()

`SqlSessionFactoryBuilder().build()`主要用来解析配置文件，`MyBatis`在解析配置文件时，会在内部创建一个`Configuration`对象来保存配置，该类位于`org.apache.ibatis.session`包下，配置文件中的所有配置项在此类中都以字段方式来存储，`Configuration `对象会在`MyBatis `初始化过程中创建且是全局唯一的，

`SqlSessionFactoryBuilder().build()`会调用多个重载，最终会调用到如下方法

```java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
    return build(parser.parse());
    //省略try-catch..
}

public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
}
```

#### XMLConfigBuilder

`XMLConfigBuilder`继承自`BaseBuilder`,`BaseBuilder`的子类如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/XMLMapperBuilder.png)



这里`MyBatis`使用了建造者模式，如果你对此模式还不是很熟悉，建议看一看[这篇文章](https://suiyueranzly.gitee.io/posts/709379917/)或者去网上看一些相关资料。`BaseBuilder`主要字段有

```java
/** 前面提到过保存配置的对象 **/
protected final Configuration configuration;
/** 保存别名的映射，之前的文章已经分析过 **/
protected final TypeAliasRegistry typeAliasRegistry;
/** 保存JdbcType和TypeHandler的映射关系，之前的文章已经分析过 **/
protected final TypeHandlerRegistry typeHandlerRegistry;
```

`BaseBuilder`中提供了一些方法以供子类使用，实现比较简单，这里不再过多分析。

回到`XMLConfigBuilder`类，该类主要字段有

```java
/** 标识是否已经解析过 **/
private boolean parsed;
/** XPathParser用来解析配置文件，之前文章已经分析过 **/
private XPathParser parser;
/** 标识当前的environment名称 **/
private String environment;
/** 负责创建Reflector对象，之前文章已经分析过 **/
private ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();
```

`MyBatis`初始化时会调用`XMLConfigBuilder.parse()`方法来解析配置文件

```java
public Configuration parse() {
    //只能解析一次
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    //以<configuration>为起点开始解析
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}
```

`parseConfiguration()`方法中，解析了`MyBatis`现支持的所有配置，每个节点的解析单独封装成一个方法，稍后会逐个分析这些方法。

```java
private void parseConfiguration(XNode root) {
    try {
        //解析<properties>配置
        propertiesElement(root.evalNode("properties"));
        //解析<settings>配置
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        //根据<settings>配置来设置Configuration对象的vfsImpl字段
        loadCustomVfs(settings);
        //解析<typeAliases>配置
        typeAliasesElement(root.evalNode("typeAliases"));
        //解析<plugins>配置
        pluginElement(root.evalNode("plugins"));
        //解析<objectFactory>配置
        objectFactoryElement(root.evalNode("objectFactory"));
        //解析<objectWrapperFactory>配置
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        //解析<reflectorFactory>配置
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        //根据<settings>配置来设置Configuration对象的对应字段
        settingsElement(settings);
        //解析<environments>配置
        environmentsElement(root.evalNode("environments"));
        //解析<databaseIdProvider>配置
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        //解析<typeHandlers>配置
        typeHandlerElement(root.evalNode("typeHandlers"));
        //解析<mappers>配置
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("..");
    }
}
```

##### 解析&lt;properties&gt;配置

使用`properties`标签可以将变量都配置在一处，在其它地方直接用占位符方式使用。

`propertiesElement()`方法会解析所有的变量并保存在`XPathParser`和`Configuration`的`variables`字段中。

```java
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        //获取标签下所有配置项
        Properties defaults = context.getChildrenAsProperties();
        //获取是否有resource或者url属性
        String resource = context.getStringAttribute("resource");
        String url = context.getStringAttribute("url");
        //两个属性只能存在一个
        if (resource != null && url != null) {
            throw new BuilderException("...");
        }
        //根据属性来加载对应的文件
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
        } else if (url != null) {
            defaults.putAll(Resources.getUrlAsProperties(url));
        }
        //获取当前已经保存的变量
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        //重新赋值
        parser.setVariables(defaults);
        configuration.setVariables(defaults);
    }
}
```

##### 解析&lt;setting&gt;配置

`settings`是 `MyBatis `中极为重要的调整设置，它们会改变` MyBatis `的运行时行为。`settingsAsProperties()`用来解析此标签

```java
private Properties settingsAsProperties(XNode context) {
    if (context == null) {
        return new Properties();
    }
    //获取该标签下所有子配置项
    Properties props = context.getChildrenAsProperties();
    // 为Configuration类创建MetaClass对象，MetaClass之前的文章已经分析过
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
        //如果Configuration对象中没有该属性，证明配置项是错的，则抛出异常
        if (!metaConfig.hasSetter(String.valueOf(key))) {
            throw new BuilderException("....");
        }
    }
    return props;
}
```

##### 配置vfsImpl

`loadCustomVfs()`会根据`settings`标签中的配置项来设置`Configuration`对象的`vfsImpl`字段

```java
private void loadCustomVfs(Properties props) throws ClassNotFoundException {
    //获取setting中vfsImpl配置项的值
    String value = props.getProperty("vfsImpl");
    if (value != null) {
        //可能是多个
        String[] clazzes = value.split(",");
        for (String clazz : clazzes) {
            if (!clazz.isEmpty()) {
                //赋值
                Class<? extends VFS> vfsImpl = (Class<? extends VFS>)Resources.classForName(clazz);
                configuration.setVfsImpl(vfsImpl);
            }
        }
    }
}
```

##### 解析&lt;typeAliases&gt;配置

类型别名是为 `Java` 类型设置一个短的名字。它只和 `XML `配置有关，存在的意义仅在于用来减少类完全限定名的冗余。

```java
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        //获取所有的子节点
        for (XNode child : parent.getChildren()) {
            //如果配置的是package，则取出包名
            if ("package".equals(child.getName())) {
                String typeAliasPackage = child.getStringAttribute("name");
                configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
            } else {
                //如果配置的是typeAlias
                //则取出别名和对应的类
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                    Class<?> clazz = Resources.classForName(type);
                    //根据别名是否为空分别注册到typeAliasRegistry
                    if (alias == null) {
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    throw new BuilderException("...");
                }
            }
        }
    }
}
```

##### 解析&lt;plugins&gt;配置

可能有些同学对此配置项有些陌生 ，下面简单介绍一下（我这里抄的是官方实例。。）

`MyBatis `允许在已映射语句执行过程中的某一点进行拦截调用，通过` MyBatis `提供的强大机制，使用插件是非常简单的，只需实现` Interceptor `接口，并指定想要拦截的方法签名即可。

```java
// ExamplePlugin.java
@Intercepts({@Signature(
    type= Executor.class,
    method = "update",
    args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
    public Object intercept(Invocation invocation) throws Throwable {
        return invocation.proceed();
    }
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
    public void setProperties(Properties properties) {
    }
}
```

然后在配置文件中加入以下配置

```xml
<!-- mybatis-config.xml -->
<plugins>
    <plugin interceptor="org.mybatis.example.ExamplePlugin">
        <property name="someProperty" value="100"/>
    </plugin>
</plugins>
```

继续说回到代码，`pluginElement()`方法用来解析`plugins`标签

```java
private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
        //获取所有子节点
        for (XNode child : parent.getChildren()) {
            //获取interceptor属性，此属性配置的是类的全限定名
            String interceptor = child.getStringAttribute("interceptor");
            //获取配置的property标签
            Properties properties = child.getChildrenAsProperties();
            //实例化配置的自定义插件类
            Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
            //对自定义插件类中的字段赋值
            interceptorInstance.setProperties(properties);
            //加入到configuration对象中
            configuration.addInterceptor(interceptorInstance);
        }
    }
}
```

##### 解析&lt;objectFactory&gt;配置

前面的文章提到过，我们可以自定义`objectFactory、objectWrapperFactory、reflectorFactory`来改变`Mybatis`的默认行为，实现对`Mybatis`的拓展。

`objectFactoryElement()`用来解析配置的自定义`objectFactory`实现类。

```java
private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
        //获取type属性，此属性是类的全限定名
        String type = context.getStringAttribute("type");
        //获取自定义实现类的字段属性配置
        Properties properties = context.getChildrenAsProperties();
        //实例化
        ObjectFactory factory = (ObjectFactory) resolveClass(type).newInstance();
        //为自定义实现类的字段赋值
        factory.setProperties(properties);
        //加入到configuration对象中
        configuration.setObjectFactory(factory);
    }
}
```

`objectWrapperFactory()\reflectorFactory()`与该方式实现类似，这里就不在过多阐述了。

##### settingsElement()

`settingsElement()`会根据`setting`标签中的配置项对`configuration`对象中对应的字段赋值，方法实现比较简单。

```java
private void settingsElement(Properties props) throws Exception {
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    //省略其它
}
```

##### 解析&lt;environments&gt;配置

`MyBatis` 可以配置成适应多种环境，这种机制有助于将 `SQL `映射应用于多种数据库之中， 现实情况下有多种理由需要这么做。例如，开发、测试和生产环境需要有不同的配置；或者共享相同 `Schema `的多个生产数据库。

`environmentsElement()`方法用来解析`environments`标签

```java
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        //如果environment属性为空，则取出默认的环境配置
        if (environment == null) {
            environment = context.getStringAttribute("default");
        }
        //取出所有的子节点，每个节点对应一个environment
        for (XNode child : context.getChildren()) {
            String id = child.getStringAttribute("id");
            //如果是当前的默认环境
            if (isSpecifiedEnvironment(id)) {
                //获取配置项来构建Environment.Builder对象
                TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
                DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
                DataSource dataSource = dsFactory.getDataSource();
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                //赋值到configuration对象中
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```

##### 解析&lt;databaseIdProvider&gt;配置

`MyBatis` 可以根据不同的数据库厂商执行不同的语句，这种多厂商的支持是基于映射语句中的 `databaseId` 属性。 `MyBatis` 会加载不带 `databaseId` 属性和带有匹配当前数据库 `databaseId` 属性的所有语句。 如果同时找到带有 `databaseId` 和不带 `databaseId` 的相同语句，则后者会被舍弃。

`databaseIdProviderElement()`方法用来解析`databaseIdProvider`标签

```java
private void databaseIdProviderElement(XNode context) throws Exception {
    DatabaseIdProvider databaseIdProvider = null;
    if (context != null) {
        //获取type属性
        String type = context.getStringAttribute("type");
        // 兼容以前版本
        if ("VENDOR".equals(type)) {
            type = "DB_VENDOR";
        }
        //获取所有子属性
        Properties properties = context.getChildrenAsProperties();
        //初始化DatabaseIdProvider对象并赋值
        databaseIdProvider = (DatabaseIdProvider) resolveClass(type).newInstance();
        databaseIdProvider.setProperties(properties);
    }
    //获取当前环境
    Environment environment = configuration.getEnvironment();
    if (environment != null && databaseIdProvider != null) {
        //设置databaseId
        String databaseId = databaseIdProvider.getDatabaseId(environment.getDataSource());
        configuration.setDatabaseId(databaseId);
    }
}
```

##### 解析&lt;typeHandlers&gt;配置

无论是 `MyBatis `在预处理语句（`PreparedStatement`）中设置一个参数时，还是从结果集中取出一个值时， 都会用类型处理器将获取的值以合适的方式转换成` Java `类型。`typeHandlers`可以配置数据库类型和`Java`类型的映射

```java
private void typeHandlerElement(XNode parent) throws Exception {
    if (parent != null) {
        //遍历所有的子节点
        for (XNode child : parent.getChildren()) {
            //如果配置的是package标签
            if ("package".equals(child.getName())) {
                //根据包名批量注册，typeHandlerRegistry.register()之前已经分析过
                String typeHandlerPackage = child.getStringAttribute("name");
                typeHandlerRegistry.register(typeHandlerPackage);
            } else {
                //如果不是package标签则取出配置的属性
                String javaTypeName = child.getStringAttribute("javaType");
                String jdbcTypeName = child.getStringAttribute("jdbcType");
                String handlerTypeName = child.getStringAttribute("handler");
                //根据配置的属性实例化出Class对象或JdbcType对象
                Class<?> javaTypeClass = resolveClass(javaTypeName);
                JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
                Class<?> typeHandlerClass = resolveClass(handlerTypeName);
                //注册到typeHandlerRegistry中
                if (javaTypeClass != null) {
                    if (jdbcType == null) {
                        typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
                    } else {
                        typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
                    }
                } else {
                    typeHandlerRegistry.register(typeHandlerClass);
                }
            }
        }
    }
}
```

##### 解析&lt;mappers&gt;配置

` MyBatis `的行为已经由上述元素配置完了，我们现在就要定义`SQL `映射语句了。但是首先我们需要告诉 `MyBatis` 到哪里去找到这些语句。 `Java `在自动查找这方面没有提供一个很好的方法，所以最佳的方式是告诉 `MyBatis` 到哪里去找映射文件。你可以使用相对于类路径的资源引用， 或完全限定资源定位符（包括 `file:///` 的 `URL`），或类名和包名等。

`mapperElement()`方法用来解析`mappers`标签

```java
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        //遍历所有子节点
        for (XNode child : parent.getChildren()) {
            //如果配置的package标签，则根据包名批量添加
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            } else {
                //如果不是package标签，则取出resource、url、class属性
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                //根据不同的属性初始化出XMLMapperBuilder对象
                //XMLMapperBuilder对象会在下一篇文章讲解
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                    mapperParser.parse();
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                    mapperParser.parse();
                } else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                } else {
                    //如果都不符合则抛出异常
                    throw new BuilderException("....");
                }
            }
        }
    }
}
```

### 结语

本篇文章讲解了`MyBatis`解析`mybatis-config.xml`配置文件的过程，下篇文章会讲解解析`Mapper.xml`文件的步骤。



































