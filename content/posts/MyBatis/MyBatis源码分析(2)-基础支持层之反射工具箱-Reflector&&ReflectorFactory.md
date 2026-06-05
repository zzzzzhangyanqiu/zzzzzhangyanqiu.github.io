---
title: MyBatis源码分析(2)-基础支持层之反射工具箱-Reflector&&ReflectorFactory
tags:
  - MyBatis
date: 2018-07-19
categories:
 - MyBatis
---

# 反射工具箱 #

`MyBatis`在进行参数处理，结果映射时会涉及到大量的反射操作。`Java`中虽然已经提供了反射，但代码略微复杂。为了简化反射相关的操作，`MyBatis`提供了专门的反射模块，位于`org.apache.ibatis.reflection`包下，它对`Java`中的反射做了进一步的封装，提供了简洁易用的`API`，本篇文章主要分析其中的`Reflector&&ReflectorFactory`。

## Reflector ##

首先我们分析的是`Reflector`，`Reflector`是`MyBatis`中反射模块的基础，一个类对应一个`Reflector`对象，此类中封装了类的各种信息，包括类元信息、字段、方法等。`Reflector`中的字段如下

```java

	private static final String[] EMPTY_STRING_ARRAY = new String[0];
		
	//保存当前反射的类
	private Class<?> type;
	//可读属性名称集合	在某种意义上来说，可读属性也就是有getXX()方法的属性
	private String[] readablePropertyNames = EMPTY_STRING_ARRAY;
	//可写属性名称集合
	private String[] writeablePropertyNames = EMPTY_STRING_ARRAY;
	//保存setter方法，key为属性名称，value为方法对象，后面会分析Invoker对象
	private Map<String, Invoker> setMethods = new HashMap<String, Invoker>();
	//保存getter方法，key为属性名称，value为方法对象
	private Map<String, Invoker> getMethods = new HashMap<String, Invoker>();
	//保存setter方法的类型，key为方法名称，value为参数类型
	private Map<String, Class<?>> setTypes = new HashMap<String, Class<?>>();
	//保存getter方法的类型,key为方法名称，value为方法返回值
	private Map<String, Class<?>> getTypes = new HashMap<String, Class<?>>();
	//保存默认构造方法，也就是没有参数的构造方法
	private Constructor<?> defaultConstructor;
	
	//保存所有属性名称的集合
	private Map<String, String> caseInsensitivePropertyMap = new HashMap<String, String>();

```


在`Reflector`的构造方法中，会进行初始化并填充上述字段，一起来看该类的构造方法

```java
  public Reflector(Class<?> clazz) {
    type = clazz;
    //查找默认的构造方法
    addDefaultConstructor(clazz);
    //查找getter方法，该方法会填充getMethods和getTypes字段
    addGetMethods(clazz);
    //查找setter方法，该方法会填充setMethods和setTypes字段
    addSetMethods(clazz);
    //查找没有getter和setter方法的字段
    addFields(clazz);
    //根据getMethods、setMethods集合填充readablePropertyNames和writeablePropertyNames字段
    readablePropertyNames = getMethods.keySet().toArray(new String[getMethods.keySet().size()]);
    writeablePropertyNames = setMethods.keySet().toArray(new String[setMethods.keySet().size()]);
    //转换大写后填充caseInsensitivePropertyMap集合，在获取时也会将key转换为大写
    for (String propName : readablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
    for (String propName : writeablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
  }
```

添加默认构造方法比较简单,就是获取所有的构造方法并且找到无参的一个

```java
  private void addDefaultConstructor(Class<?> clazz) {
    //获取所有的构造方法
    Constructor<?>[] consts = clazz.getDeclaredConstructors();
    //循环找到无参的方法并赋值
    for (Constructor<?> constructor : consts) {
      if (constructor.getParameterTypes().length == 0) {
        if (canAccessPrivateMethods()) {
          try {
            constructor.setAccessible(true);
          } catch (Exception e) {
            // Ignored. This is only a final precaution, nothing we can do.
          }
        }
        if (constructor.isAccessible()) {
          this.defaultConstructor = constructor;
        }
      }
    }
  }
```

下面我们来看添加`addGetMethods`和`addSetMethods`方法，由于两个方法实现较为相似，这里只分析`addGetMethods`

```java
  private void addGetMethods(Class<?> cls) {
    Map<String, List<Method>> conflictingGetters = new HashMap<String, List<Method>>();
    //获取该类和父类中所有的方法对应的method对象
    Method[] methods = getClassMethods(cls);
    for (Method method : methods) {
      String name = method.getName();
      //如果是get方法并且长度大于三（避免方法名只是get()，没有属性名称）
      if (name.startsWith("get") && name.length() > 3) {
        if (method.getParameterTypes().length == 0) {
          /*
          * 调用PropertyNamer.methodToProperty方法，通过substring获取属性名称并且首字母小写
          * 如getAbc()  这里就是abc     isBcd()  这里就是bcd
          * */
          name = PropertyNamer.methodToProperty(name);
          /*
          * 将method对象和属性名称记录到conflictingGetters集合中
          * 注意，conflictingGetters的value是list类型的，具体原因后面会解释
          * */
          addMethodConflict(conflictingGetters, name, method);
        }
      } else if (name.startsWith("is") && name.length() > 2) {
        //处理is方法
        if (method.getParameterTypes().length == 0) {
          name = PropertyNamer.methodToProperty(name);
          addMethodConflict(conflictingGetters, name, method);
        }
      }
    }
    /*
    * 之前说过，conflictingGetters的value是list类型的，原因如下
    * 当子类重写了父类的getter方法并修改了返回值类型的时候
    * 在addUniqueMethods()方法中就会产生两条数据
    * 如现有类A 及其子类SubA, A 类中定义了getNames()方法，其返回值类型是List<String>，
    * 而在其子类SubA 中， 覆写了其getNames()方法且将返回值修改成ArrayList<String>
    * 最终得到的两个方法签名分别是java.util.List#getNames和java.util.ArrayList#getNames
    * 所以conflictingGetters方法的value为list类型
    * resolveGetterConflicts就是要处理这种情况
    * */
    resolveGetterConflicts(conflictingGetters);
  }
```

可能你在此方法中看到了很多陌生的方法，先别着急，下面我们来分析这些陌生的方法，一点一点的揭开它们的“面纱”，首先来看`getClassMethods`，该方法是获取`Class`对象的所有方法（包含父类方法）

```java
  private Method[] getClassMethods(Class<?> cls) {
    Map<String, Method> uniqueMethods = new HashMap<String, Method>();
    Class<?> currentClass = cls;
    while (currentClass != null) {
      //为每个方法生成唯一签名，记录到uniqueMethods集合中
      addUniqueMethods(uniqueMethods, currentClass.getDeclaredMethods());

      /*
      * 记录接口中定义的方法，因为这个类可能是一个抽象类或者接口
      * */
      Class<?>[] interfaces = currentClass.getInterfaces();
      for (Class<?> anInterface : interfaces) {
        addUniqueMethods(uniqueMethods, anInterface.getMethods());
      }

      //继续查找父类方法
      currentClass = currentClass.getSuperclass();
    }

    //将结果返回
    Collection<Method> methods = uniqueMethods.values();

    return methods.toArray(new Method[methods.size()]);
  }

  private void addUniqueMethods(Map<String, Method> uniqueMethods, Method[] methods) {
    for (Method currentMethod : methods) {
      //如果方法不是桥接方法
      if (!currentMethod.isBridge()) {
        /*
        * 为方法生成签名，签名规则为  返回值类型#方法名称:参数类型列表1.参数类型列表2
        * 如 public String User.getUserId(String userId, Integer userId1)的方法签名为
        * java.lang.String#getSignature:java.lang.String,java.lang.Integer
        * */
        String signature = getSignature(currentMethod);
        /*
        * 检测是否已经在子类添加过此方法了，如果已经添加过则不再添加
        * */
        if (!uniqueMethods.containsKey(signature)) {
          if (canAccessPrivateMethods()) {
            try {
              currentMethod.setAccessible(true);
            } catch (Exception e) {
              // Ignored. This is only a final precaution, nothing we can do.
            }
          }

          //返回结果
          uniqueMethods.put(signature, currentMethod);
        }
      }
    }
  }


```


`getSignature()`中获取了方法的签名，将签名做为唯一标识加入到集合中


```java
  private String getSignature(Method method) {
    StringBuilder sb = new StringBuilder();
    Class<?> returnType = method.getReturnType();
    //如果有返回值则拼接  返回值#
    if (returnType != null) {
      sb.append(returnType.getName()).append('#');
    }
    //拼接方法名称
    sb.append(method.getName());
    //拼接方法的参数类型
    Class<?>[] parameters = method.getParameterTypes();
    for (int i = 0; i < parameters.length; i++) {
      //如果是第一个则拼接”:参数类型“  非首个则拼接 “,参数类型”
      if (i == 0) {
        sb.append(':');
      } else {
        sb.append(',');
      }
      sb.append(parameters[i].getName());
    }
    //返回结果
    return sb.toString();
  }
```

继续回到`addGetMethods()`方法。`getClassMethods()`已经了解过了，我们继续往下面分析。`PropertyNamer.methodToProperty()`方法比较简单，就是从方法名中提取出属性的名称。

```java
  public static String methodToProperty(String name) {
    //如果是is开头，则从第二位开始截取
    if (name.startsWith("is")) {
      name = name.substring(2);
    } else if (name.startsWith("get") || name.startsWith("set")) {
      //如果是get或者set开头，则从第三位开始截取
      name = name.substring(3);
    } else {
      //都不是则抛出异常
      throw new ReflectionException("Error parsing property name '" + name + "'.  Didn't start with 'is', 'get' or 'set'.");
    }

    //因为getName这种方法，截取出来的属性名首字母是大写的，所以要把第一位转换成小写
    if (name.length() == 1 || (name.length() > 1 && !Character.isUpperCase(name.charAt(1)))) {
      name = name.substring(0, 1).toLowerCase(Locale.ENGLISH) + name.substring(1);
    }

    return name;
  }
```

`addMethodConflict()`会为属性找到最合适的`getter`方法

```java
  private void resolveGetterConflicts(Map<String, List<Method>> conflictingGetters) {
    for (String propName : conflictingGetters.keySet()) {
      //获取属性对应的多个getter方法
      List<Method> getters = conflictingGetters.get(propName);
      Iterator<Method> iterator = getters.iterator();
      Method firstMethod = iterator.next();
      //如果只有一个getter方法，证明没有发生上面说的情况，直接添加get方法即可
      if (getters.size() == 1) {
        addGetMethod(propName, firstMethod);
      } else {
        Method getter = firstMethod;
        //记录返回值类型
        Class<?> getterType = firstMethod.getReturnType();
        while (iterator.hasNext()) {
          Method method = iterator.next();
          Class<?> methodType = method.getReturnType();
          //如果返回值相同，这种情况应该在addUniqueMethods()方法中被过滤掉，这里直接抛出异常
          if (methodType.equals(getterType)) {
            throw new ReflectionException("Illegal overloaded getter method with ambiguous type for property "
                + propName + " in class " + firstMethod.getDeclaringClass()
                + ".  This breaks the JavaBeans " + "specification and can cause unpredictable results.");
          } else if (methodType.isAssignableFrom(getterType)) {
            // 如果当前方法返回值是第一个方法返回值的父类，那么第一个方法还是最合适的，什么也不做
          } else if (getterType.isAssignableFrom(methodType)) {
            // 如果当前方法返回值是第一个方法返回值的子类，则当前方法是最合适的
            getter = method;
            getterType = methodType;
          } else {
            // 如果还有其他情况直接抛出异常
            throw new ReflectionException("Illegal overloaded getter method with ambiguous type for property "
                + propName + " in class " + firstMethod.getDeclaringClass()
                + ".  This breaks the JavaBeans " + "specification and can cause unpredictable results.");
          }
        }
        //添加到getMethods、getTypes字段中
        addGetMethod(propName, getter);
      }
    }
  }
```

看到这里可能你已经比较清楚了，所谓的解决同属性名多个方法之间的冲突，就是保存最合适的一个，也就是参数类型级别最小的一个。例如上面说的举的例子，`setName(String)`方法和`setName(Object object)`，因为`String`级别要比`Object`低，所以就会保留`setName(String)`。下面我们来分析`addGetMethod()`

```java
  private void addGetMethod(String name, Method method) {
    //校验属性名是否合法
    if (isValidPropertyName(name)) {
      //填充getMethods和getTypes集合
      getMethods.put(name, new MethodInvoker(method));
      //解析方法返回值类型加入到getTypes中，TypeParameterResolver将在下篇文章中详细讲解
      Type returnType = TypeParameterResolver.resolveReturnType(method, type);
      getTypes.put(name, typeToClass(returnType));
    }
  }

  private boolean isValidPropertyName(String name) {
    return !(name.startsWith("$") || "serialVersionUID".equals(name) || "class".equals(name));
  }
```

此方法的逻辑就是将方法和返回值加入到对应的集合之中，需要注意的是，`MyBatis`是将`method`对象封装成了`Invoker`对象。一起来看`Invoker`接口，`Invoker`接口有三个实现类，分别是：`GetFieldInvoker`提供属性`getter()`方法的封装、`MethodInvoker`提供`Method`对象的封装以及`SetFieldInvoker`提供属性`setter()`方法的封装，这三个类实现的比较简单，这里就不在深入分析了，对应类图如下


![](/images/oss/%E5%8D%9A%E5%AE%A2/Invoker%E7%B1%BB%E5%9B%BE.png)



回到`Reflector`的构造方法，还剩下一个方法，那就是`addFields()`，该方法的作用是查找没有`getter`和`setter`方法的字段

```java
  private void addFields(Class<?> clazz) {
    //获取该类声明的字段
    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
      if (canAccessPrivateMethods()) {
        try {
          field.setAccessible(true);
        } catch (Exception e) {
          // Ignored. This is only a final precaution, nothing we can do.
        }
      }
      if (field.isAccessible()) {
        //如果setMethods集合中不包含同名属性，也就是该字段没有setter方法
        if (!setMethods.containsKey(field.getName())) {
          // 过滤掉final和static修饰的字段
          int modifiers = field.getModifiers();
          if (!(Modifier.isFinal(modifiers) && Modifier.isStatic(modifiers))) {
            //添加到setMethods、setTypes集合中
            addSetField(field);
          }
        }
        //如果getMethods集合中不包含同名属性，也就是该字段没有getter方法
        if (!getMethods.containsKey(field.getName())) {
          //添加到getMethods、getTypes集合中
          addGetField(field);
        }
      }
    }
    //继续查找父类
    if (clazz.getSuperclass() != null) {
      addFields(clazz.getSuperclass());
    }
  }
```



## ReflectorFactory ##

`ReflectorFactory`提供了对`Reflector`对象的创建及缓存，该接口有三个方法

```java
public interface ReflectorFactory {

  /**
   * 是否开启缓存
   * **/
  boolean isClassCacheEnabled();

  /**
   * 设置缓存开启状态
   * **/
  void setClassCacheEnabled(boolean classCacheEnabled);

  /**
   * 为class对象创建对应的Reflector对象
   * **/
  Reflector findForClass(Class<?> type);
}
```

`MyBatis`只为`ReflectorFactory`提供了一个实现类`DefaultReflectorFactory`，该类的主要字段和方法如下

```java
  /**
   * 默认开启缓存
   * */
  private boolean classCacheEnabled = true;
  /**
   * 缓存用到的map集合
   * */
  private final ConcurrentMap<Class<?>, Reflector> reflectorMap = new ConcurrentHashMap<Class<?>, Reflector>();

  public Reflector findForClass(Class<?> type) {
    //如果开启了缓存功能
    if (classCacheEnabled) {
      /*
      * 查看map集合中是否已经有了缓存的内容，如果有则直接返回
      * 如果没有则创建后加入缓存并返回
      * */
      Reflector cached = reflectorMap.get(type);
      if (cached == null) {
        cached = new Reflector(type);
        reflectorMap.put(type, cached);
      }
      return cached;
    } else {
      //如果没有开启缓存则直接新建
      return new Reflector(type);
    }
  }
```


除了`MyBatis`为我们提供了默认的实现类，还可以通过配置文件配置自定义的实现类，从而改变`MyBatis`的行为，增加拓展功能，后面会提到此

