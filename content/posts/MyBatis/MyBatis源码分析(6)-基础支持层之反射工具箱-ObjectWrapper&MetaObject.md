---
title: MyBatis源码分析(6)-基础支持层之反射工具箱-ObjectWrapper&MetaObject
tags:
  - MyBatis
date: 2018-08-26
categories:
 - MyBatis
---



### ObjectWrapper

上篇文章我们了解了`MetaClass`，主要作用是处理一些类级别的信息。这篇文章我们来了解`ObjectWrapper&MetaObject`，这两个类因为作用大体相同，所以放在一起记录。此类主要是处理一些对象级别的信息，如设置对象的属性值，获取对象的属性值等等。首先来看一下该类包含的方法

```java
public interface ObjectWrapper {

  //获取表达式对应的值
  Object get(PropertyTokenizer prop);

  //根据表达式赋值
  void set(PropertyTokenizer prop, Object value);

  //如果属性存在则返回属性值，如果不存在则返回null,第二个参数为是否忽略属性表达式的下划线
  String findProperty(String name, boolean useCamelCaseMapping);

  //获取可读属性集合
  String[] getGetterNames();

  //获取可写属性集合
  String[] getSetterNames();

  //返回setter方法的参数类型
  Class<?> getSetterType(String name);

  //返回getter方法的返回值类型
  Class<?> getGetterType(String name);

  //属性表达式是否拥有setter/getter方法
  boolean hasSetter(String name);

  boolean hasGetter(String name);

  //为表达式指定的属性创建MetaObject对象
  MetaObject instantiatePropertyValue(String name, PropertyTokenizer prop, ObjectFactory objectFactory);

  //是否是集合
  boolean isCollection();

  //对应集合中的add/addAll方法
  void add(Object element);
  
  <E> void addAll(List<E> element);

}
```

首先我们看一下创建该接口实现类的工厂类`ObjectWrapperFactory`，该接口只有两个方法

```java
  boolean hasWrapperFor(Object object);

  ObjectWrapper getWrapperFor(MetaObject metaObject, Object object);
```

`MyBatis`中为我们提供了一个默认的实现类`DefaultObjectWrapperFactory`，需要注意的是，该类中的方法一个是返回`return false`，一个是直接抛出异常。也就是说，该实现类实际上是不可用的。我们需要自己实现该类，并通过修改配置来改变`MyBatis`的默认行为，这点以后会提到。

继续回到`ObjectWrapper`中，该接口有两个实现类，其中一个实现类中还包含两个子类，整体类关系图如下

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/ObjectWrapper%E7%B1%BB%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

由于整体逻辑类似，这里只记录`BaseWrapper`和`BeanWrapper`两个类。了解了这两个类之后，再去看其它两个类也是游刃有余了。

#### BaseWrapper

此类是一个抽象类。只提供了几个供子类使用的方法，也就是说，其余的各个接口要依靠子类自己实现，该类依赖于`MetaObject`，包括其此类也要依靠该类实现一些功能，此类我们稍后讲解。`BaseWrapper`中只有方法`resolveCollection()`、`getCollectionValue()`、`setCollectionValue()`，由于`getCollectionValue()/setCollectionValue()`逻辑比较类似，所以这里只分析`resolveCollection()/getCollectionValue()`两个方法，有兴趣的同学可以在源码中自行查看其它方法。

首先来看下代码

```java
protected Object resolveCollection(PropertyTokenizer prop, Object object) {
    //如果表达式名称为空则直接返回object对象
    if ("".equals(prop.getName())) {
        return object;
    } else {
        //如果不为空则调用metaObject.getValue方法获取对应值
        return metaObject.getValue(prop.getName());
    }
}  
protected Object getCollectionValue(PropertyTokenizer prop, Object collection) {
    if (collection instanceof Map) {
        //如果是map集合，则index为key
        return ((Map) collection).get(prop.getIndex());
    } else {
        //如果是其它集合，则index为数组下标
        int i = Integer.parseInt(prop.getIndex());
        if (collection instanceof List) {
            return ((List) collection).get(i);
        } else if (collection instanceof Object[]) {
            return ((Object[]) collection)[i];
        } else if (collection instanceof char[]) {
            //其它类型的数组省略。。。
        } else {
            throw new ReflectionException("。。。");
        }
    }
}
```

两个方法比较简单，这里就不再过多阐述，下面我们一起来看一下其子类。

#### BeanWrapper

由于`BeanWrapper`的父类`BaseWrapper`没有实现一个方法，所以`ObjectWrapper`接口中的方法全都在此类中实现。不过大多数的方法都是依赖`MetaClass/MetaObject`类来完成的，这里就不分析其它方法了。我们只分析`get()/set()`两个较为核心的方法，同样，由于两个方法实现逻辑比较类似，我们这里只分析`get()`方法。

```java

private Object object;
private MetaClass metaClass;

@Override
public Object get(PropertyTokenizer prop) {
    //如果包含索引则调用解析集合方法
    if (prop.getIndex() != null) {
        //这里调用的是父类BaseWrapper的两个方法，这里不在阐述
        Object collection = resolveCollection(prop, object);
        return getCollectionValue(prop, collection);
    } else {
        //如果不包含则直接调用解析属性方法
        return getBeanProperty(prop, object);
    }
}


  private Object getBeanProperty(PropertyTokenizer prop, Object object) {
    try {
      //获取属性的Invoker对象
      Invoker method = metaClass.getGetInvoker(prop.getName());
      try {
        //获取返回值类型
        return method.invoke(object, NO_ARGUMENTS);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    } catch (RuntimeException e) {
      throw e;
    } catch (Throwable t) {
      throw new ReflectionException("。。。");
    }
  }
```

这里一定要充分的理解`get()`方法，之后会和`MetaObject`递归调用，如不理解此方法，后面可能会越来越乱。

#### MetaObject

通过上面的了解，`ObjectWrapper`类的主要作用是获取/设置或者解析对象中的属性，之前已经提过，`ObjectWrapper`中的大部分功能都要依赖于`MetaClass/MetaObject`来完成。像是上文中介绍的`BeanWrapper.get()/BeanWrapper.set()`，`BeanWrapper`只是解析最后结果，整个解析过程还是在`MetaObject`中来实现的。下面一起看下代码

##### 创建

```java
//原始对象
private Object originalObject;

//原始对象所属的ObjectWrapper对象
private ObjectWrapper objectWrapper;

//创建originalObject的factory工厂对象
private ObjectFactory objectFactory;

//创建ObjectWrapper的factory工厂对象
private ObjectWrapperFactory objectWrapperFactory;

//创建Reflector的factory工厂对象
private ReflectorFactory reflectorFactory;

private MetaObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
    this.originalObject = object;
    this.objectFactory = objectFactory;
    this.objectWrapperFactory = objectWrapperFactory;
    this.reflectorFactory = reflectorFactory;

    //如果已经是ObjectWrapper对象，则直接拿过来用就可以了
    if (object instanceof ObjectWrapper) {
        this.objectWrapper = (ObjectWrapper) object;
    } else if (objectWrapperFactory.hasWrapperFor(object)) {
        //如果objectWrapperFactory已经存在了此对象，则直接拿出来用
        this.objectWrapper = objectWrapperFactory.getWrapperFor(this, object);
    } else if (object instanceof Map) {
        //如果是map集合，则转换为MapWrapper对象
        this.objectWrapper = new MapWrapper(this, (Map) object);
    } else if (object instanceof Collection) {
        //如果是collection集合，则转换为CollectionWrapper对象
        this.objectWrapper = new CollectionWrapper(this, (Collection) object);
    } else {
        //如果都不是，则直接转换为BeanWrapper对象
        this.objectWrapper = new BeanWrapper(this, object);
    }
}
```

`MetaObject`中，构造方法是`private`的，不过此类提供了一个静态的方法来创建该对象

```java
public static MetaObject forObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
    if (object == null) {
        //如果为null则直接返回空对象
        return SystemMetaObject.NULL_META_OBJECT;
    } else {
        //如果不为空则调用构造方法
        return new MetaObject(object, objectFactory, objectWrapperFactory, reflectorFactory);
    }
}
```

##### 使用

此类中比较核心的方法有三个`getValue()/setValue()/metaObjectForProperty()`，分别是获取值，设置值和根据属性表达式获取相应的`metaObject`对象。先来看一下`metaObjectForProperty()`方法。

```java
public MetaObject metaObjectForProperty(String name) {
    //根据表达式获取值
    Object value = getValue(name);
    //调用创建方法
    return MetaObject.forObject(value, objectFactory, objectWrapperFactory, reflectorFactory);
}
```

由于`getValue()/setValue()`两个方法实现逻辑类似，这里只介绍`getValue()`

```java
public Object getValue(String name) {
    PropertyTokenizer prop = new PropertyTokenizer(name);
    //如果有子级表达式（就是带“.”的，如user.id，id就是子级表达式）,
    if (prop.hasNext()) {
        //首先获取父级的MetaObject对象
        MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
        if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
            return null;
        } else {
            //递归调用获取
            return metaValue.getValue(prop.getChildren());
        }
    } else {
        //如果没有子级表达式(不带“.”的，如id，name)，就直接调用objectWrapper.get()方法
        return objectWrapper.get(prop);
    }
}
```

为了方便理解，这里举例说明，考虑以下的情况

```java
public class RichType {
    
    private List<GenericConcrete> genericConcretes = new ArrayList<GenericConcrete>(){
        {
            GenericConcrete genericConcrete1 = new GenericConcrete();
            genericConcrete1.setId(1L);
            add(genericConcrete1);
        }
    };
}
```

然后我们编写一段测试代码

```java
//这里的参数为"genericConcretes[0].id"，
//解析成PropertyTokenizer后name为"genericConcretes"
//indexedName为"genericConcretes[0]"
//index为"0"
//children为"id"
String name = "genericConcretes[0].id";
RichType rich = new RichType();
MetaObject meta = SystemMetaObject.forObject(rich);
System.out.println(meta.getValue(name));
```

以上代码输出结果为

`1`

下面我会用文字来叙述解析过程，帮助理解

1. 首先`MetaObject.getValue("genericConcretes[0].id")`方法判断出有子级表达式，调用`metaObjectForProperty()`方法来解析`genericConcretes[0]`获取`genericConcretes`中第一个元素的`MetaObject`对象。
2. `metaObjectForProperty()`方法会调用`MetaObject.getValue("genericConcretes[0]")`来解析集合数组，注意此时的参数是包含索引的。
3. `MetaObject.getValue("genericConcretes[0]")`判断出没有子级表达式，会调用`objectWrapper.get("genericConcretes[0]")`方法来解析集合数据，而`objectWrapper.get("genericConcretes[0]")`方法判断表达式中包含索引，会调用`resolveCollection()`和`getCollectionValue()`方法来解析集合数据，并返回对应索引的对象。
4. 获取到对象后，一路返回到`metaObjectForProperty()`方法，此方法会为集合中对应索引的对象创建一个`MetaObject`对象返回，`MetaObject.getValue()`方法获取到此`MetaObject`对象会，再调用此对象的`metaValue.getValue("id")`方法，参数为子级表达式。
5. `metaValue.getValue("id")`方法中判断没有子集表达式，直接调用`objectWrapper.get("id")`，该方法调用`objectWrapper.getBeanProperty`方法直接获取`id`的值。至此，全部解析过程结束。



这两个类中每个方法看起来较为简单，复杂的是相互的递归调用，理解了这里，其它方法就比较简单了。







