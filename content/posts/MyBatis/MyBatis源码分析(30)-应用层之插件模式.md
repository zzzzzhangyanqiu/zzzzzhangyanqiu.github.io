---
title: MyBatis源码分析(30)-应用层之插件模式
tags:
  - MyBatis
date: 2019-05-11
categories:
 - MyBatis
---

### 插件模式

为了便于拓展，`MyBatis`中为我们提供了插件的模式。虽然叫插件，却是使用`Interceptor`-拦截器的方式实现的。

`MyBatis` 允许在已映射语句执行过程中的某一点进行拦截调用。默认情况下，`MyBatis` 允许拦截的方法调用包括：

- `Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)`

- `ParameterHandler (getParameterObject, setParameters)`
- `ResultSetHandler (handleResultSets, handleOutputParameters)`
- `StatementHandler (prepare, parameterize, batch, update, query)`

`MyBatis`中提供的插件功能比较强大，可以使我们改变`MyBatis`的核心处理逻辑，比如分页插件，分库分表插件等等。正因为如此，所以在编写拦截器之前，要深入的了解`MyBatis`，这样才能编写出便于使用的插件，`MyBatis`的插件模块中使用了责任链模式，如果你还不是很熟悉此设计模式，建议看一下[这篇文章](https://suiyueranzly.gitee.io/posts/3066120463/)或者去网上找一些相关资料，下面我们一起看一下插件的实现过程

#### Interceptor

首先我们需要自定义拦截器，实现`org.apache.ibatis.plugin.Interceptor`接口，并为该类添加`@Intercepts`注解，该注解会配置`@Signature`类型的数组，`@Signature`注解可以配置三个参数

`type`：拦截的类型，`method`：拦截的方法名称，`args`：拦截的方法参数

定义好拦截器之后，将自定义的类配置到`mybatis-config.xml`中，就可以实现插件的功能了。

举例说明，如下代码会拦截`Executor.query()`方法

```java
@Intercepts({@Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class,RowBounds.class, ResultHandler.class})})
public class Interceptor implements org.apache.ibatis.plugin.Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("这里拦截");
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```

然后在`mybatis-config.xml`中配置

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/MyBatis/interceptor%E9%85%8D%E7%BD%AE.png)

这样就可以进行自定义拦截逻辑了

简单介绍一下此处实现的三个方法

`intercept()`是拦截器的主要逻辑，方法最后面的`invocation.proceed()`会执行原逻辑

`plugin()`用来返回原执行类的代理类，可以考虑使用`MyBatis`为我们提供的`Plugin.wrap()`方法完成，说到这里，可能大家会有点想法，在设置了拦截器后，`MyBatis`会返回原始类的`代理类`，以此实现拦截的功能，该实现逻辑后面会详细分析

`setProperties()`方法中可执行用户自己的逻辑，其中的`properties`参数为用户在`mybatis-config.xml`的`plugin`标签下自定义的属性（`property`标签）

下面来分析插件模块的实现原理

首先需要我们了解的是`MyBatis`在解析`mybatis-config.xml`配置文件时，会将`plugins`下配置的自定义拦截器添加到`Configuration.interceptorChain()`字段存储，各位可以回顾以前的文章

`MyBatis`在创建上述的四个可拦截类时，调用的都是`Configuration.new*()`方法，下面以`newExecutor()`进行举例分析

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    //根据不同的ExecutorType创建不同的执行器
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }
    //如果开启了二级缓存，则添加CachingExecutor作为执行器
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    //进行自定义拦截器的处理
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

`InterceptorChain`中只有一个`private final List<Interceptor> interceptors`字段，该字段存储的是所有用户自定义的拦截器，在`InterceptorChain.pluginAll()`中，会依次处理这些自定义的拦截器

```java
public Object pluginAll(Object target) {
    //遍历并调用自定义拦截器的plugin()方法
    for (Interceptor interceptor : interceptors) {
        target = interceptor.plugin(target);
    }
    return target;
}
```

我们前面说过，在`plugin()`方法中推荐使用`Plugin.wrap()`方法，该方法会为目标类创建动态代理类

```java
public static Object wrap(Object target, Interceptor interceptor) {
    //获取Signature注解的相关信息，其中key为Signature的type属性，也就是要拦截的类
    //value为对应的方法
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    //获取所有接口，创建动态代理时会用到
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
        //创建动态代理
        return Proxy.newProxyInstance(
            type.getClassLoader(),
            interfaces,
            new Plugin(target, interceptor, signatureMap)); //代理类为Plugin
    }
    return target;
}
```

`Plugin`类的字段如下

```java
/**
   * 目标类
   * **/
private Object target;
/***
   * 拦截器
   * **/
private Interceptor interceptor;
/***
   * 拦截的相关信息
   * key为Signature的type属性，也就是要拦截的类
   * value为对应的方法
   * **/
private Map<Class<?>, Set<Method>> signatureMap;
```

既然使用`Plugin`作为代理类，我们主要关注的肯定就是`invoke()`方法

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        //获取该类需要拦截的方法
        Set<Method> methods = signatureMap.get(method.getDeclaringClass());
        //判断是否需要拦截，执行不同的逻辑
        if (methods != null && methods.contains(method)) {
            return interceptor.intercept(new Invocation(target, method, args));
        }
        return method.invoke(target, args);
    } catch (Exception e) {
        throw ExceptionUtil.unwrapThrowable(e);
    }
}
```

`Invocation`中记录了当前调用的一些基本信息，主要字段如下

```java
/***
   * 目标类，也就是代理之前的原始类
   * **/
private Object target;
/***
   * 被拦截的方法
   * **/
private Method method;
/***
   *
   * 被拦截方法的参数
   * **/
private Object[] args;
```

`Invocation.proceed()`方法则会调用`method.invoke()`执行原方法逻辑



