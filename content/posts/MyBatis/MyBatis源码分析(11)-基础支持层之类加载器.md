---
title: MyBatis源码分析(11)-基础支持层之类加载器
tags:
  - MyBatis
date: 2018-10-31
categories:
 - MyBatis
---

## 类加载器 

`MyBatis`中提供了`ClassLoaderWrapper`来进行资源的加载，该类的逻辑是调用其内部封装的`ClassLoader`来进行各种方式的加载操作，如果你对`Java`中的`ClassLoader`来不是很熟悉，建议先在网上看一些相关资料。

### ClassLoaderWrapper

`MyBatis`提供的`ClassLoaderWrapper`是一个`ClassLoader`的包装器，其内部封装了多个`ClassLoader`对象。在资源加载时候，会依次检测内部封装的`ClassLoader`，并从中选择正确的对象返回。`ClassLoaderWrapper`有两个核心字段

```java
//系统默认的ClassLoader
ClassLoader defaultClassLoader;
//Java中的SystemAppClassLoader（也被称为AppclassLoader）
ClassLoader systemClassLoader;
```

在构造方法中会初始化`systemClassLoader`字段

```java
ClassLoaderWrapper() {
    //省略try-catch
    systemClassLoader = ClassLoader.getSystemClassLoader();
}
```

`ClassLoaderWrapper`中的方法主要分三类，分别是`getResourceAsURL()/getResourceAsStream()/classForName()`，这三个方法有多个重载，最后都会调用参数为`(String , ClassLoader[])`的重载。三个方法实现逻辑类似，这里以`getResourceAsURL()`为例进行说明

```java
public URL getResourceAsURL(String resource) {
    //调用重载
    return getResourceAsURL(resource, getClassLoaders(null));
}

public URL getResourceAsURL(String resource, ClassLoader classLoader) {
    //调用重载
    return getResourceAsURL(resource, getClassLoaders(classLoader));
}

URL getResourceAsURL(String resource, ClassLoader[] classLoader) {

    URL url;

    //遍历传入的多个ClassLoader
    for (ClassLoader cl : classLoader) {

        //如果不为空
        if (null != cl) {

            // 调用ClassLoader.getResource()方法查找资源
            url = cl.getResource(resource);

            // 如果没有找到在前面拼接“/”再次查找
            if (null == url) {
                url = cl.getResource("/" + resource);
            }

            //如果找到则直接返回，没有找到则进行下次循环
            if (null != url) {
                return url;
            }

        }

    }

    //没有找到则返回空
    return null;

}
```

`getClassLoaders()`会加载相关的`ClassLoaders`并返回

```java
ClassLoader[] getClassLoaders(ClassLoader classLoader) {
    return new ClassLoader[]{
        //传入的ClassLoader
        classLoader,
        //系统默认的ClassLoader
        defaultClassLoader,
        //当前线程绑定的ClassLoader
        Thread.currentThread().getContextClassLoader(),
        //加载当前类的ClassLoader
        getClass().getClassLoader(),
        //Java的AppclassLoader
        systemClassLoader};
}
```

`org.apache.ibatis.io.Resources`类是一个工具类，其中封装了多个访问资源的工具方法，大多数都是调用`ClassLoaderWrapper`中的方法，代码比较简单，请读者自行查看。

 