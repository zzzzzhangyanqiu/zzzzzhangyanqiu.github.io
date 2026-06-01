---
title: MyBatis源码分析(8)-基础支持层之别名配置
tags:
  - MyBatis
date: 2018-09-15
categories:
 - MyBatis
---



## 别名配置

在我们使用`SQL`语句时，如果表名或者列名很长时，一般我们会为其设置一个别名来方便操作。延续了`SQL`的这种思想，`MyBatis`提供了对类配置别名的功能，这样就可以通过别名来引用该类。

### TypeAliasRegistry

`MyBatis`通过`TypeAliasRegistry`类来完成类的别名注册。该类结构比较简单，内部有一个`Map<String, Class<?>> TYPE_ALIASES`集合来存储别名和类之间的关系，并且在该类初始化时，就已经为`Java`中的基础类型、包装类型、数组类型、集合等创建了别名，如下

```java
public TypeAliasRegistry() {
    registerAlias("string", String.class);
    registerAlias("byte", Byte.class);
    registerAlias("long", Long.class);
    registerAlias("short", Short.class);
    
    //省略其它..
}
```

#### registerAlias()

`TypeAliasRegistry.registerAlias(String alias, Class<?> value)`方法提供了将别名注册到指定类的功能

```java
public void registerAlias(String alias, Class<?> value) {
    //如果别名为空则直接抛出异常
    if (alias == null) {
        throw new TypeException("...");
    }
    //首先将别名统一转换为小写
    String key = alias.toLowerCase(Locale.ENGLISH);
    //如果已经被注册过了，则抛出异常
    if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
        throw new TypeException("...");
    }
    //注册到集合中
    TYPE_ALIASES.put(key, value);
}
```

#### registerAliases()

`TypeAliasRegistry.registerAlias(String packageName, Class<?> superType)`提供了批量扫描指定包下的所有类，并为指定类的子类添加别名功能。

```java
public void registerAliases(String packageName, Class<?> superType){
    //扫描指定包下指定类的所有子类
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for(Class<?> type : typeSet){
        //跳过匿名类、接口和内部类
        if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
            //注册该类
            registerAlias(type);
        }
    }
}

public void registerAlias(Class<?> type) {
    //获取类的简单名字(不包含包名)
    String alias = type.getSimpleName();
    //如果有@Alias注解则取值，没有就以类的简单名字做为key来注册
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
        alias = aliasAnnotation.value();
    } 
    registerAlias(alias, type);
}
```

#### resolveAlias()

`TypeAliasRegistry.registerAliasresolveAlias(String string)`可以根据传入的别名找到注册的类。

```java
public <T> Class<T> resolveAlias(String string) {
    try {
        //如果别名为空则直接返回空
        if (string == null) {
            return null;
        }
        //将别名转换为小写
        String key = string.toLowerCase(Locale.ENGLISH);
        Class<T> value;
        //如果包含则直接取出
        if (TYPE_ALIASES.containsKey(key)) {
            value = (Class<T>) TYPE_ALIASES.get(key);
        } else {
            //如果不包含则尝试直接加载该类
            value = (Class<T>) Resources.classForName(string);
        }
        return value;
    } catch (ClassNotFoundException e) {
        throw new TypeException("...");
    }
}
```

































