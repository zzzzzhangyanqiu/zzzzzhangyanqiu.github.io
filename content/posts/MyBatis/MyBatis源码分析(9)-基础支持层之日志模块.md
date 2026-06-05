---
title: MyBatis源码分析(9)-基础支持层之日志模块
tags:
  - MyBatis
date: 2018-10-13
categories:
 - MyBatis
---

## 日志模块

日志在软件中扮演了必不可少的重要角色，良好的日志记录可以帮助开发和运维人员快速的定位问题，从而加速线上问题的结局，但是日志会随着系统的运行时间增长而不断增长。所以，定期汇总和清理日志是一个良好的行为。在`Java` 开发中常用的日志框架有`Log4j` 、`Log4j2` 、`Apache Commons Log `、`java.util.logging`、`slf4j`等，这些框架的接口各不相同。为了统一这些框架的接口， `MyBatis `定义了一套统一的日志接口供上层使用， 并为上述常用的日志框架提供了相应的适配器。`MyBatis `的日志模块中，使用了适配器模式，如果你对此模式还不了解，建议先看一下之前的[文章](https://suiyueranzly.gitee.io/posts/554442395/)或者看看相关的资料。`MyBatis `调用日志模块时，会调用其内部的适配器接口`org.apache.ibatis.logging.Log`，再由此接口调用各日志框架的API。这样，`MyBatis `就可以统一的使用各日志框架了。

### 日志适配器

前面描述的多种第三方日志组件都有各自的`Log `级别，且都有所不同，例如`java.util.logging`
提供了`All` 、`FINEST` 、`FINER` 、`FINE `、`CONFIG `、`INFO` 、`WARNING` 等9 种级别，而`Log4j2` 则
只有`trace` 、`debug `、`info `、`warn` 、`error`、`fatal `这6 种日志级别。`MyBatis` 统一提供了`trace` 、`debug `、`warn` 、`error` 四个级别，这基本与主流日志框架的日志级别类似，可以满足绝大多数场景的日志
需求。

`MyBatis`日志模块位于`org.apache.ibatis.logging`包中，通过`Log`接口提供了日志的功能，再由各个适配器统一实现接口。`LogFactory`负责创建`Log`对象，类关系图如下。

![](/images/oss/%E5%8D%9A%E5%AE%A2/Log%E7%B1%BB%E5%9B%BE.png)



#### Log

首先来看一下`Log`接口的子类和代码

![](/images/oss/%E5%8D%9A%E5%AE%A2/log%E5%AD%90%E7%B1%BB.png)

```java
public interface Log {

    /**
   * 是否开启了Debug
   * **/
    boolean isDebugEnabled();

    /**
   * 是否开启了Trace
   * **/
    boolean isTraceEnabled();

    /**
   * 日志的各个级别
   * **/
    void error(String s, Throwable e);

    void error(String s);

    void debug(String s);

    void trace(String s);

    void warn(String s);

}
```

#### LogFactory

`LogFactory`类主要的工作是创建相应的日志适配器类，`LogFactory`初始化时会执行`static`代码快的内容，主要工作是加载各个日志适配器，通过`Constructor<? extends Log> logConstructor`保存当前日志适配器的初始化方法。

```java
/**
   * 记录了当前日志适配器的初始化方法
   * **/
private static Constructor<? extends Log> logConstructor;

static {
    //以下会针对每种日志组件调用tryImplementation进行加载，具体调用顺序为
    //Slf4j>Commons>Log4J2>Log4J>JdkLog>NoLog
    tryImplementation(new Runnable() {
        @Override
        public void run() {
            //使用Slf4j框架
            useSlf4jLogging();
        }
    });
    tryImplementation(new Runnable() {
        @Override
        public void run() {
            //使用Commons日志框架
            useJdkLogging();
        }
    });
    //其它方法省略。。
}

```

`tryImplementation()`会判断当前的`logConstructor`是否为空，然后执行`runnable.run()`方法，方法实现比较简单，这里就不在阐述。下面一起来分析`useJdkLogging()`方法，其它方法实现逻辑类似。

```java
public static synchronized void useJdkLogging() {
    //Jdk14LoggingImpl是MyBatis提供的针对jdkLog的日志适配器类
    setImplementation(org.apache.ibatis.logging.jdk14.Jdk14LoggingImpl.class);
}

private static void setImplementation(Class<? extends Log> implClass) {
    try {
        //获取构造方法
        Constructor<? extends Log> candidate = implClass.getConstructor(String.class);
        //实例化该日志适配器
        Log log = candidate.newInstance(LogFactory.class.getName());
        //打印日志
        if (log.isDebugEnabled()) {
            log.debug("...");
        }
        //赋值
        logConstructor = candidate;
    } catch (Throwable t) {
        throw new LogException("...");
    }
}
```

然后就是日志适配器的实现类了，同理，这里也以`jdkLog`来举例，`Mybatis`为`jdkLog`提供的适配器是`Jdk14LoggingImpl`。

#### Jdk14LoggingImpl

```java
public class Jdk14LoggingImpl implements Log {

    //java.util.logging.Logger
    private Logger log;

    //获取jdkLog
    public Jdk14LoggingImpl(String clazz) {
        log = Logger.getLogger(clazz);
    }

    @Override
    public boolean isDebugEnabled() {
        return log.isLoggable(Level.FINE);
    }

    @Override
    public boolean isTraceEnabled() {
        return log.isLoggable(Level.FINER);
    }

    @Override
    public void error(String s, Throwable e) {
        log.log(Level.SEVERE, s, e);
    }

	//其它级别方法省略。。

}
```

如果你了解适配器模式，这里就非常简单了，其思想就是使用适配器模式调用`jdkLog`的方法。其余日志适配器类的实现逻辑类似，感兴趣可自行查看源码。





















