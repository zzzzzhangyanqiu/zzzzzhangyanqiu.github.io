---
title: MyBatis源码分析(24)-核心处理层之延迟加载
tags:
  - MyBatis
date: 2019-04-21
categories:
 - MyBatis
---

### 延迟加载

延迟加载，意为暂时不用的对象不会加载到内存中，直到需要的时候才会去查询数据库并加载。在`MyBatis`进行属性映射时，如果遇到需要延迟加载的对象或属性，则首先会创建延迟加载的代理对象，当调用此属性或对象时，就会调用代理对象的相关方法来完成查询并加载到内存中，这里用到了代理模式。

与延迟加载有关的属性主要有以下几个

| 属性名称                 | 说明                                                         | 默认值                                       |
| :----------------------- | ------------------------------------------------------------ | -------------------------------------------- |
| `lazyLoadingEnabled`     | 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。 特定关联关系中可通过设置 `resultMap`标签中的`fetchType` 属性来覆盖该项的开关状态。 | false                                        |
| `aggressiveLazyLoading`  | 当开启时，任何方法的调用都会加载该对象的所有属性。 否则，每个属性会按需加载 | false （在 3.4.1 及之前的版本默认值为 true） |
| `lazyLoadTriggerMethods` | 指定哪个方法触发一次延迟加载。                               | `equals,clone,hashCode,toString`             |

前面说到，`MyBatis`是通过代理模式来实现延迟加载，关于代理模式，可以查看[这篇文章](https://suiyueranzly.gitee.io/posts/3822496321/)或者去网上看一些相关资料。

除文章中说到的方式外，`MyBatis`还用到了另外一种技术`Javassist`

#### Javassist

`Javassist`是一个开源的分析、编辑和创建`Java`[字节码](https://baike.baidu.com/item/字节码)的类库。`javassist`是`jboss`的一个子项目，其主要的优点，在于简单，而且快速。直接使用`java`编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构，或者动态生成类。简单来说，就是可以用`java`代码的方式创建一个类。

首先来了解基本用法

```java
ClassPool classPool = ClassPool.getDefault();
//类名
CtClass ctClass = classPool.makeClass("com.javassist.Test");
//创建字段
CtField field = new CtField(CtClass.intType, "age", ctClass);
//使用private 修饰
field.setModifiers(AccessFlag.PRIVATE);
//设置 age字段的 get/set方法
ctClass.addMethod(CtNewMethod.setter("setAge", field));
ctClass.addMethod(CtNewMethod.getter("getAge", field));
//设置字段初始值，并添加到类中
ctClass.addField(field, CtField.Initializer.constant(12));
//创建构造方法
CtConstructor constructor = CtNewConstructor.make("public Test(){" +
                                                  "this.age = 15;" +
                                                  "System.out.println(\"constructor()\");" +
                                                  "}", ctClass);
//添加构造方法
ctClass.addConstructor(constructor);
//创建方法
CtMethod method = CtNewMethod.make("public void showAge(){" +
                                   "System.out.println(\"showAge():\" + this.age);" +
                                   "}", ctClass);
//添加方法
ctClass.addMethod(method);
//将定义的class保存到磁盘
ctClass.writeFile("d:/");


//测试上面创建的类
Class aClass = ctClass.toClass();
Object o = aClass.newInstance();
//调用showAge方法
Method showAge = o.getClass().getMethod("showAge");
showAge.invoke(o);
```

上述代码运行后，会在`D:\com\javassist`下生成`Test.class`文件，并在控制台输出

```java
constructor()
showAge():15
```

打开`Test.class`文件，其内容为

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.javassist;

public class Test {
    private int age = 12;

    public void setAge(int var1) {
        this.age = var1;
    }

    public int getAge() {
        return this.age;
    }

    public Test() {
        this.age = 15;
        System.out.println("constructor()");
    }

    public void showAge() {
        System.out.println("showAge():" + this.age);
    }
}
```

了解了基本使用后，我们一起来看如何使用`Javassist`实现动态代理

```java
public class ShopDao {

    public void add() {
        System.out.println("增加商品");
    }

    public void remove() {
        System.out.println("移除商品");
    }
}

public class Test {
    public static void main(String[] args) throws IOException {
        try {
            test();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void test() throws Exception {
        ClassPool classPool = ClassPool.getDefault();
        //类名
        CtClass ctClass = classPool.makeClass("com.javassist.Test");
        //创建字段
        CtField field = new CtField(CtClass.intType, "age", ctClass);
        //使用private 修饰
        field.setModifiers(AccessFlag.PRIVATE);
        //设置 age字段的 get/set方法
        ctClass.addMethod(CtNewMethod.setter("setAge", field));
        ctClass.addMethod(CtNewMethod.getter("getAge", field));
        //设置字段初始值，并添加到类中
        ctClass.addField(field, CtField.Initializer.constant(12));
        //创建构造方法
        CtConstructor constructor = CtNewConstructor.make("public Test(){" +
                                                          "this.age = 15;" +
                                                          "System.out.println(\"constructor()\");" +
                                                          "}", ctClass);
        //添加构造方法
        ctClass.addConstructor(constructor);
        //创建方法
        CtMethod method = CtNewMethod.make("public void showAge(){" +
                                           "System.out.println(\"showAge():\" + this.age);" +
                                           "}", ctClass);
        //添加方法
        ctClass.addMethod(method);
        //将定义的class保存到磁盘
        ctClass.writeFile("d:/");


        //测试上面创建的类
        Class aClass = ctClass.toClass();
        Object o = aClass.newInstance();
        //调用showAge方法
        Method showAge = o.getClass().getMethod("showAge");
        showAge.invoke(o);

    }
}
```

以上代码执行结果为

```java
前置操作
增加商品
后置操作
移除商品
```

#### ResultLoader&ResultLoaderMap

`MyBatis`中，与延迟加载有关的类有`ResultLoader、ResultLoaderMap`、`ProxyFactory`接口及其实现类

首先来介绍`ResultLoader`

##### ResultLoader

`ResultLoader`负责具体延迟加载的操作，其主要字段如下

```java
//关联的Configuration对象
protected final Configuration configuration;
//数据库执行对象，以后会详细讲解，目前只知道它可以执行sql语句就行了，此对象负责延迟加载
protected final Executor executor;
//记录延迟加载的sql
protected final MappedStatement mappedStatement;
//记录延迟加载SQL的参数
protected final Object parameterObject;
//记录延迟加载的类型
protected final Class<?> targetType;
//用来创建java对象
protected final ObjectFactory objectFactory;
//cacheKey在前面已经分析过了
protected final CacheKey cacheKey;
//记录延迟加载的sql
protected final BoundSql boundSql;
//此对象用来将延迟加载的到的结果转换为targetType类型的对象
protected final ResultExtractor resultExtractor;
//得到当前对象的线程ID
protected final long creatorThreadId;
//是否已经加载过了
protected boolean loaded;
//延迟加载得到的对象
protected Object resultObject;
```

`ResultLoader`的核心方法是`loadResult()`，该方法负责加载`SQL`语句，并得到结果对象

```java
public Object loadResult() throws SQLException {
    //使用sql语句进行查询
    List<Object> list = selectList();
    //将得到的结果转换为targetType类型的对象
    resultObject = resultExtractor.extractObjectFromList(list, targetType);
    return resultObject;
}
```

`selectList()`负责查询

```java
private <E> List<E> selectList() throws SQLException {
    Executor localExecutor = executor;
    //如果当前线程没有创建过Executor对象，或者Executor对象已经是关闭状态，则重新创建
    if (Thread.currentThread().getId() != this.creatorThreadId || localExecutor.isClosed()) {
        localExecutor = newExecutor();
    }
    try {
        //查询并返回结果
        return localExecutor.<E> query(mappedStatement, parameterObject, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER, cacheKey, boundSql);
    } finally {
        if (localExecutor != executor) {
            //关闭Executor对象
            localExecutor.close(false);
        }
    }
}
```

`ResultExtractor.extractObjectFromList()`负责将查询到的结果转换为`targetType`类型的对象，分为以下几种情况

```java
public Object extractObjectFromList(List<Object> list, Class<?> targetType) {
    Object value = null;
    //情况1：如果结果为list类型，则直接返回
    if (targetType != null && targetType.isAssignableFrom(list.getClass())) {
        value = list;
    } else if (targetType != null && objectFactory.isCollection(targetType)) {
        //情况2：如果结果是Collection类型，则使用objectFactory创建相应对象
        // 并调用addAll方法将其全部添加到value集合中
        value = objectFactory.create(targetType);
        MetaObject metaObject = configuration.newMetaObject(value);
        metaObject.addAll(list);
    } else if (targetType != null && targetType.isArray()) {
        //情况3：如果是数组类型，则将value转换为数组
        Class<?> arrayComponentType = targetType.getComponentType();
        Object array = Array.newInstance(arrayComponentType, list.size());
        if (arrayComponentType.isPrimitive()) {
            for (int i = 0; i < list.size(); i++) {
                Array.set(array, i, list.get(i));
            }
            value = array;
        } else {
            value = list.toArray((Object[])array);
        }
    } else {
        //如果判断到了这里，还是有多个结果，则抛出异常
        if (list != null && list.size() > 1) {
            throw new ExecutorException("Statement returned more than one row, where no more than one was expected.");
        } else if (list != null && list.size() == 1) {
            //返回第一个对象
            value = list.get(0);
        }
    }
    return value;
}
```

##### ResultLoaderMap

`ResultLoaderMap`用来保存字段与其对应的`ResultLoader`，通过以下字段

```java
//保存列名与其对应的ResultLoader，key为转换为大写的属性名
private final Map<String, LoadPair> loaderMap = new HashMap<String, LoadPair>();
```

`LoadPair`是`ResultLoaderMap`的一个内部类，其主要字段如下

```java
/**
     * 设置加载属性的MetaObject对象
     */
private transient MetaObject metaResultObject;
/**
     * 用于延迟加载的ResultLoader对象
     */
private transient ResultLoader resultLoader;
/**
     * 日志
     */
private transient Log log;
/**
     * 延迟加载的属性名称
     */
private String property;
/**
     * 加载属性的SQL语句ID
     */
private String mappedStatement;
/**
     * SQL语句的参数
     */
private Serializable mappedParameter;
```

`ResultLoaderMap`使用`load()`和`loadAll()`方法进行属性的延迟加载，不同的是前者是加载单个属性，后者是加载全部属性

```java
public void loadAll() throws SQLException {
    //取出所有延迟加载的属性，调用load方法
    final Set<String> methodNameSet = loaderMap.keySet();
    String[] methodNames = methodNameSet.toArray(new String[methodNameSet.size()]);
    for (String methodName : methodNames) {
        load(methodName);
    }
}

public boolean load(String property) throws SQLException {
    //调用LoadPair.load()方法
    LoadPair pair = loaderMap.remove(property.toUpperCase(Locale.ENGLISH));
    if (pair != null) {
        pair.load();
        return true;
    }
    return false;
}
```

可以看到，两个方法最终都是调用的`LoadPair.load()`

```java
public void load() throws SQLException {
    //校验合法性
    if (this.metaResultObject == null) {
        throw new IllegalArgumentException("metaResultObject is null");
    }
    if (this.resultLoader == null) {
        throw new IllegalArgumentException("resultLoader is null");
    }

    this.load(null);
}


public void load(final Object userObject) throws SQLException {
    //忽略部分代码
    //获取Configuration对象
    final Configuration config = this.getConfiguration();
    //获取相关sql语句
    final MappedStatement ms = config.getMappedStatement(this.mappedStatement);

    this.metaResultObject = config.newMetaObject(userObject);
    //创建ResultLoader对象
    this.resultLoader = new ResultLoader(config, new ClosedExecutor(), ms, this.mappedParameter,
                                         metaResultObject.getSetterType(this.property), null, null);
    //调用resultLoader.loadResult()方法加载属性
    this.metaResultObject.setValue(property, this.resultLoader.loadResult());

}
```

#### ProxyFactory

`ProxyFactory`负责创建代理对象，它有两个实现类，分别是`CglibProxyFactory`和`JavassistProxyFactory`

分别以`cglib`和`javassist`方法创建代理类，类图如下

![](/images/oss/%E5%8D%9A%E5%AE%A2/MyBatis/proxyfactory%E7%B1%BB%E5%9B%BE.png)

由于两个类实现逻辑类似，这里只分析`CglibProxyFactory`

##### CglibProxyFactory

`CglibProxyFactory`初始化时会尝试加载`cglib`包，如果加载失败则抛出异常

```java
public CglibProxyFactory() {
    try {
        Resources.classForName("net.sf.cglib.proxy.Enhancer");
    } catch (Throwable e) {
        throw new IllegalStateException("Cannot enable lazy loading because CGLIB is not available. Add CGLIB to your classpath.", e);
    }
}
```

`CglibProxyFactory`核心方法是`createProxy()`，该方法会调用`EnhancedResultObjectProxyImpl.createProxy()`，`EnhancedResultObjectProxyImpl`是`CglibProxyFactory`的内部类，和前面我们分析的`cglib`方式实现动态代理一样，该类也实现了`MethodInterceptor`，字段如下

```java
//代理的目标类
private final Class<?> type;
//ResultLoaderMap对象
private final ResultLoaderMap lazyLoader;
//对应配置文件中的aggressiveLazyLoading值
private final boolean aggressive;
//触发延迟加载的方法
private final Set<String> lazyLoadTriggerMethods;
private final ObjectFactory objectFactory;
//构造方法参数类型集合
private final List<Class<?>> constructorArgTypes;
//构造方法参数集合
private final List<Object> constructorArgs;
```

既然实现了`MethodInterceptor`，那其核心方法就一定是`intercept()`

```java
public Object intercept(Object enhanced, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    final String methodName = method.getName();
    try {
        synchronized (lazyLoader) {
            //对于writeReplace方法的特殊处理...
            } else {
                //如果不是finalize方法
                if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
                    //判断是否全部进行延迟加载
                    if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                        //加载全部
                        lazyLoader.loadAll();
                    } else if (PropertyNamer.isGetter(methodName)) {
                        //如果是getter方法
                        final String property = PropertyNamer.methodToProperty(methodName);
                        //加载此属性
                        if (lazyLoader.hasLoader(property)) {
                            lazyLoader.load(property);
                        }
                    }
                }
            }
        }
        return methodProxy.invokeSuper(enhanced, args);
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}
}
```

回到`EnhancedResultObjectProxyImpl.createProxy()`方法

该方法会实例化`EnhancedResultObjectProxyImpl`自身并作为`CallBack`参数调用`CglibProxyFactory.crateProxy()`方法创建代理类

```java
public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    final Class<?> type = target.getClass();
    //实例化
    EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
    //调用CglibProxyFactory.crateProxy()
    Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);
    //拷贝属性
    PropertyCopier.copyBeanProperties(type, target, enhanced);
    return enhanced;
}
```

如果已经了解了`cglib`，那`CglibProxyFactory.crateProxy()`应该就会很熟悉了，该方法实现如下

```java
static Object crateProxy(Class<?> type, Callback callback, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    //创建Enhancer接口并设置代理类
    Enhancer enhancer = new Enhancer();
    enhancer.setCallback(callback);
    enhancer.setSuperclass(type);
	//省略处理writeReplace方法
    Object enhanced;
    //如果构造方法参数类型列表为空，证明存在无参的构造方法，直接创建
    if (constructorArgTypes.isEmpty()) {
        enhanced = enhancer.create();
    } else {
        Class<?>[] typesArray = constructorArgTypes.toArray(new Class[constructorArgTypes.size()]);
        Object[] valuesArray = constructorArgs.toArray(new Object[constructorArgs.size()]);
        enhanced = enhancer.create(typesArray, valuesArray);
    }
    return enhanced;
}
```

#### 回顾

下面简单回顾一下前面的文章中提到的关于嵌套查询和延迟加载的代码片段，首先是`DefaultResultSetHandler.createParameterizedResultObject()`，该方法中有一句

```java
//处理嵌套查询
if (constructorMapping.getNestedQueryId() != null) {
    value = getNestedQueryConstructorValue(rsw.getResultSet(), constructorMapping, columnPrefix);
} 
```

如果`contructor`节点存在嵌套查询，则调用`getNestedQueryConstructorValue()`处理嵌套查询，该方法实现如下

```java
private Object getNestedQueryConstructorValue(ResultSet rs, ResultMapping constructorMapping, String columnPrefix) throws SQLException {
    //获取查询sql
    final String nestedQueryId = constructorMapping.getNestedQueryId();
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    //获取参数类型和实参
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, constructorMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
        //获取该嵌套查询对应的BoundSql和CacheKey对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        //获取该嵌套查询的结果对象
        final Class<?> targetType = constructorMapping.getJavaType();
        //创建ResultLoader加载结果
        final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
        value = resultLoader.loadResult();
    }
    return value;
}
```

这里可以看到，`contructor`节点中的嵌套查询无论配置如何都不会被延迟加载

然后是`DefaultResultSetHandler.getPropertyMappingValue()`方法，该方法中关于嵌套映射的代码片段如下

```java
if (propertyMapping.getNestedQueryId() != null) {
    return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
} 
```

如果`resultMap`中的`property`节点中存在嵌套查询时，会调用`getNestedQueryMappingValue()`处理

```java
private Object getNestedQueryMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
    throws SQLException {
    //获取嵌套查询的sql和属性名
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    final String property = propertyMapping.getProperty();
    final MappedStatement nestedQuery = configuration.getMappedStatement(nestedQueryId);
    //获取参数类型和参数列表
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    final Object nestedQueryParameterObject = prepareParameterForNestedQuery(rs, propertyMapping, nestedQueryParameterType, columnPrefix);
    Object value = null;
    if (nestedQueryParameterObject != null) {
        //获取对应的BoundSql和CacheKey对象
        final BoundSql nestedBoundSql = nestedQuery.getBoundSql(nestedQueryParameterObject);
        final CacheKey key = executor.createCacheKey(nestedQuery, nestedQueryParameterObject, RowBounds.DEFAULT, nestedBoundSql);
        //获取嵌套查询后的结果类型
        final Class<?> targetType = propertyMapping.getJavaType();
        //如果该嵌套查询已经执行过
        if (executor.isCached(nestedQuery, key)) {
            //创建deferLoad对象，并通过deferLoad对象直接从缓存中加载结果
            //关于executor和deferLoad相关以后进行分析
            executor.deferLoad(nestedQuery, metaResultObject, property, key, targetType);
            //返回特殊标记
            value = DEFERED;
        } else {
            //创建ResultLoader
            final ResultLoader resultLoader = new ResultLoader(configuration, executor, nestedQuery, nestedQueryParameterObject, targetType, key, nestedBoundSql);
            if (propertyMapping.isLazy()) {
                //如果是延迟加载则返回特殊标记
                lazyLoader.addLoader(property, metaResultObject, resultLoader);
                value = DEFERED;
            } else {
                //直接进行结果加载
                value = resultLoader.loadResult();
            }
        }
    }
    return value;
}
```

另外一个方法就是`DefaultResultSetHandler.createResultObject()`，该方法涉及到嵌套查询如下

```java
// 如果存在嵌套查询，则创建出嵌套对象
if (propertyMapping.getNestedQueryId() != null && propertyMapping.isLazy()) {
    //使用ProxyFactory创建代理工厂，默认为JavassistProxyFactory
    resultObject = configuration.getProxyFactory().createProxy(resultObject, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);
    break;
}
```



有关于延迟加载和嵌套查询的内容，至此就已经全部讲解完成了





