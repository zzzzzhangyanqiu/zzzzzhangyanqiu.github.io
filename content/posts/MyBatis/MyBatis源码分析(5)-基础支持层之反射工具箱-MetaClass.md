---
title: MyBatis源码分析(5)-基础支持层之反射工具箱-MetaClass
tags:
  - MyBatis
date: 2018-08-12
categories:
 - MyBatis
---



## MetaClass

本篇内容我们主要介绍`MyBatis`中的`MetaClass`类，该类主要作用是利用`Reflector`类和`PropertyTokenizer`类实现对复杂表达式的解析。对这两个类还不熟悉的可以查看以前的文章。首先来看一下该类的字段及构造函数

```java
//Reflector的缓存类
private ReflectorFactory reflectorFactory;
//保存了类的元信息
private Reflector reflector;

private MetaClass(Class<?> type, ReflectorFactory reflectorFactory) {
    this.reflectorFactory = reflectorFactory;
    this.reflector = reflectorFactory.findForClass(type);
}

public static MetaClass forClass(Class<?> type, ReflectorFactory reflectorFactory) {
    return new MetaClass(type, reflectorFactory);
}
```

根据代码可以看出，该类不允许显示的创建，而是提供了`MetaClass.forClass()`方法获取对象。

由于各个方法介绍内容会比较多，所以我们分开介绍，首先看一下使用到的实体类

```java
public class RichType {

  private RichType richType;

  private String richField;

  private String richProperty;

  private Map richMap = new HashMap();
    
  private GenericConcrete genericConcrete;

  private List<GenericConcrete> genericConcretes = new ArrayList<GenericConcrete>(){
    {
      add(new GenericConcrete());
    }
  };

  private List richList = new ArrayList() {
    {
      add("bar");
    }
  };

  //省略getter  setter方法
}

public class GenericConcrete{
  private Long id;

  //省略getter  setter
}
```

### findProperty

该方法主要是用来解析一些复杂的表达式，如下

```java
ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
MetaClass meta = MetaClass.forClass(RichType.class, reflectorFactory);
System.out.printf("findProperty-genericConcrete.id:%s\n",meta.findProperty("genericConcrete.id"));
//查找一个不存在的属性
System.out.printf("findProperty-aaabbbccc:%s\n",meta.findProperty("aaabbbccc"));

```

输出结果为

```java
findProperty-richType.richField:richType.richField
findProperty-aaabbbccc:null
```

如果属性存在则返回属性值，如果不存在则返回null，下面一起来看该方法的实现

```java
  /***
   * 该方法主要是调用buildProperty来解析属性
   * **/
  public String findProperty(String name) {
    StringBuilder prop = buildProperty(name, new StringBuilder());
    return prop.length() > 0 ? prop.toString() : null;
  }

  /***
   * 此方法大致解析原理为：通过PropertyTokenizer来解析。如果有下一个，则递归调用此方法
   * 如果没有，则直接通过reflector获取对应属性名
   * **/
  private StringBuilder buildProperty(String name, StringBuilder builder) {
    //使用PropertyTokenizer进行解析
    PropertyTokenizer prop = new PropertyTokenizer(name);
    //如果有下一个
    if (prop.hasNext()) {
      //通过reflector获取对应属性名	如果是前面测试的代码，这里是genericConcrete
      String propertyName = reflector.findPropertyName(prop.getName());
      if (propertyName != null) {
        //如果不等于空则拼接
        builder.append(propertyName);
        builder.append(".");
        //根据属性解析出MetaClass对象
        //这里创建genericConcrete所属的MetaClass对象，然后继续解析id
        MetaClass metaProp = metaClassForProperty(propertyName);
        //开始递归调用，解析剩下的属性
        //prop.getChildren()为id
        metaProp.buildProperty(prop.getChildren(), builder);
      }
    } else {
      //如果没有下一个则直接拼接
      String propertyName = reflector.findPropertyName(name);
      if (propertyName != null) {
        builder.append(propertyName);
      }
    }
    return builder;
  }
```

需要注意的是，此方法只会解析`"."`的复杂表达式，并不会处理索引。下面一起来看一下`metaClassForProperty()`方法，该方法的主要作用是利用`reflector`根据属性名解析出`MetaClass`对象。此方法在接下来的调用次数较多，请认真阅读。

```java
  public MetaClass metaClassForProperty(String name) {
    Class<?> propType = reflector.getGetterType(name);
    return MetaClass.forClass(propType, reflectorFactory);
  }
```



### hasSetter()/hasGetter()

这两个方法分别用来判断给定的属性名是否有`getter()`和`setter()`方法。需要注意的是，由于`Reflector.addFields()`方法会给没有`getter()`和`setter()`方法的属性添加`GetFieldInvoker、SetFieldInvoker`对象，所以此方法可能不会像方法名表达的那样只是单纯的判断`getter()`和`setter()`

首先来看一下运行效果

```java
ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
MetaClass meta = MetaClass.forClass(RichType.class, reflectorFactory);
//这里就算是richField没有getter方法，只要有这个字段就会返回true，原因在前面已经讲过
System.out.printf("hasGetter-richField:%s\n",meta.hasGetter("richField"));
System.out.printf("hasGetter-richType.richField:%s\n",meta.hasGetter("richType.richField"));
//查找一个不存在的属性
System.out.printf("hasGetter-aaabbbccc:%s\n",meta.hasGetter("aaabbbccc"));
```

运行结果为

```java
hasGetter-richField:true
hasGetter-richType.richField:true
hasGetter-aaabbbccc:false
```

由于两个方法实现逻辑比较类似，这里只分析`hasGetter()`方法，有兴趣的读者可以自行查看源码。

```java
public boolean hasGetter(String name) {
    //解析表达式，我们这里拿"richType.richField"来举例分析
    PropertyTokenizer prop = new PropertyTokenizer(name);
    //如果还有子集
    if (prop.hasNext()) {
        //如果当前字段有getter方法才查找子集，例子中为richType
        if (reflector.hasGetter(prop.getName())) {
            //查找到richType对应MetaClass对象
            MetaClass metaProp = metaClassForProperty(prop);
            //递归的入口，只要存在子集，就不断向下取，一直到没有子集为止
            return metaProp.hasGetter(prop.getChildren());
        } else {
            return false;
        }
    } else {
        //如果没有子集则直接判断
        return reflector.hasGetter(prop.getName());
    }
}
```



### getGetterType()/getSetterType()

接下来是此类中较为复杂的方法`getGetterType()/getSetterType()`，由于两个方法实现逻辑比较类似，这里只介绍`getGetterType()`，首先看一下运行效果

```java
ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
MetaClass meta = MetaClass.forClass(RichType.class, reflectorFactory);
System.out.printf("getGetterType-genericConcretes:%s\n",meta.getGetterType("genericConcretes"));
System.out.printf("getGetterType-genericConcretes[0]:%s\n",meta.getGetterType("genericConcretes[0]"));
System.out.printf("getGetterType-genericConcretes[0].id:%s\n",meta.getGetterType("genericConcretes[0].id"));
```

此代码运行结果为

```java
getGetterType-genericConcretes:interface java.util.List
getGetterType-genericConcretes[0]:class org.apache.ibatis.domain.misc.generics.GenericConcrete
getGetterType-genericConcretes[0].id:class java.lang.Long
```

下面开始分析，由于解析集合内元素如`genericConcretes[0]`与直接解析属性逻辑不同，这里我们先分析直接解析属性

```java
public Class<?> getGetterType(String name) {
    //首先调用PropertyTokenizer解析表达式，例如“genericConcretes[0].id”
    PropertyTokenizer prop = new PropertyTokenizer(name);
    //如果有子集，例子中子集为“id”
    if (prop.hasNext()) {
        //根据属性获取MetaClass对象
        MetaClass metaProp = metaClassForProperty(prop);
        //开始递归调用，prop.getChildren()为“id”
        return metaProp.getGetterType(prop.getChildren());
    }
    //解析集合内元素，此方法稍后会分析
    return getGetterType(prop);
}

```

可以看到，解析`"."`的表达式时逻辑与其它方法类似，利用`PropertyTokenizer`解析表达式，然后再根据属性获取`MetaClass`对象，最后递归调用即可。接下来我们分析解析集合内的属性。

```java
private Class<?> getGetterType(PropertyTokenizer prop) {
    //这里以“genericConcretes[0]”举例
    //type为interface java.util.List
    Class<?> type = reflector.getGetterType(prop.getName());
    //如果索引值不等于空并且是集合类型
    //此处索引值为0，判断条件成立
    if (prop.getIndex() != null && Collection.class.isAssignableFrom(type)) {
        //获取属性的Type类型，这里是ParameterizedTypeImpl，此方法稍后会分析
        Type returnType = getGenericGetterType(prop.getName());
        if (returnType instanceof ParameterizedType) {
            //获取到泛型中的真实类型，此处为GenericConcrete
            Type[] actualTypeArguments = ((ParameterizedType) returnType).getActualTypeArguments();
            //判断如果不等于空并且长度等于1，由于Java中方法只能由一个返回值，
            //如果是Map<GenericConcrete,PropertyTokenizer>类型的泛型，就会不知道解析哪个类型
            if (actualTypeArguments != null && actualTypeArguments.length == 1) {
                //取出第一个
                returnType = actualTypeArguments[0];
                //如果是Class类型则直接返回
                //这里是returnType为GenericConcrete
                if (returnType instanceof Class) {
                    type = (Class<?>) returnType;
                } else if (returnType instanceof ParameterizedType) {
                    //如果是ParameterizedType则取出原始类型返回
                    type = (Class<?>) ((ParameterizedType) returnType).getRawType();
                }
            }
        }
    }
    return type;
}

```

最后我们来分析一下`getGenericGetterType()`

```java
private Type getGenericGetterType(String propertyName) {
    try {
        //为了方便理解，我们继续沿用上面的例子，这里的propertyName为genericConcretes
        //首先获取Invoker对象，由于此字段并没有getter方法，是由Reflector。addFields()
        //添加进去的，所以这里是GetFieldInvoker
        Invoker invoker = reflector.getGetInvoker(propertyName);
        if (invoker instanceof MethodInvoker) {
            //如果是MethodInvoker则获取method字段
            Field _method = MethodInvoker.class.getDeclaredField("method");
            _method.setAccessible(true);
            //获取该属性的getter方法对象
            Method method = (Method) _method.get(invoker);
            //解析方法的返回值
            return TypeParameterResolver.resolveReturnType(method, reflector.getType());
        } else if (invoker instanceof GetFieldInvoker) {
            //如果是GetFieldInvoker则获取field字段
            Field _field = GetFieldInvoker.class.getDeclaredField("field");
            _field.setAccessible(true);
            //获取该属性的field对象
            Field field = (Field) _field.get(invoker);
            //解析字段的类型
            return TypeParameterResolver.resolveFieldType(field, reflector.getType());
        }
    } catch (NoSuchFieldException e) {
    } catch (IllegalAccessException e) {
    }
    return null;
}
```

### 总结

根据各个方法可以看出，该类主要的思想就是使用`Reflector`和`PropertyTokenizer`进行各种复杂表达式的解析，如果对两个类印象不太清楚的同学可以看一下前面的文章。该类还有其它方法，比较简单，基本都是使用`Reflector`实现的，这里就不再分析了，有兴趣的可以自行查看源码。



