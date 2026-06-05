---
title: MyBatis源码分析(3)-基础支持层之反射工具箱-TypeParameterResolver
tags:
  - MyBatis
date: 2018-07-29
categories:
 - MyBatis
---



## TypeParameterResolver

​	我们上篇文章中讲到`MyBatis`中反射的基础  `Reflector` ，其中的很多方法都调用到了`TypeParameterResolver`来处理方法的返回值、方法参数或字段的类型，本篇文章将分析此类。

​	如果你对`Java`中的`Type`接口还不是很熟悉，建议先看一下[这篇文章](https://suiyueranzly.gitee.io/posts/2895841313/)来快速了解一下`Java`中的`Type`，下面我们就开始分析吧。

​	`Reflector`类中用到`TypeParameterResolver`的主要有四处，分别是`addGetMethod()`中调用 `TypeParameterResolver.resolveReturnType()` 用来解析方法返回值类型、`addSetMethod()`方法中调用`TypeParameterResolver.resolveParamTypes()` 用来解析方法参数类型、`addSetField()` 和 `addGetField()` 方法调用了`TypeParameterResolver.resolveFieldType()`用来解析字段类型。这里可能有些乱，我们通过下面的一幅图全面的来了解`TypeParameterResolver`中各方法的调用关系。

​	![](/images/oss/%E5%8D%9A%E5%AE%A2/TypeParameterResolver%E5%90%84%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8%E5%85%B3%E7%B3%BB.png)



可以看到，三个方面基本实现逻辑相同，这里我们只分析其中一个方法`resolveFieldType()`，其它方法有兴趣可以自行学习，此类可能比较难以理解，我们通过一个测试类一步一步来分析此方法。

首先创建一个`Calculator`，并创建它的子类`SubCalculator`

```java
/***
*由于TypeParameterResolver主要用来解析泛型，所以这里直接定义一个泛型类
**/
public class Calculator<T> {
  protected T id;

  private T fld;

  private List<T> list;

  protected T attribute;

  public T getId() {
    return id;
  }

  public void setId(T id) {
    this.id = id;
  }

  public static class SubCalculator extends Calculator<String> {
  }
}
```

测试方法

```java
Class<?> clazz = SubCalculator.class;
Class<?> declaredClass = Calculator.class;
Field field = declaredClass.getDeclaredField("list");
//此处的第一个参数是要待解析的属性也就是field对象
//第二个参数是开始查找的位置，本例中我们从子类中开始查找
Type result = TypeParameterResolver.resolveFieldType(field, clazz);
System.out.println(result);
System.out.println(result.getClass().getName());
```

首先看一下输出结果，再进行具体分析

```java
ParameterizedTypeImpl [rawType=interface java.util.List, ownerType=null, actualTypeArguments=[class java.lang.String]]
//注意此处的类是MyBatis自己实现的
org.apache.ibatis.reflection.TypeParameterResolver$ParameterizedTypeImpl
```

下面我们开始来具体分析代码，首先来看`resolveFieldType()`，通过名称可以得知，此方法的作用是解析字段类型

```java
public static Type resolveFieldType(Field field, Type srcType) {
    //获取属性类型  例子的类型是java.util.List<T>
    Type fieldType = field.getGenericType();
    //获取声明此属性的类 ，由于list变量是在Calculator中声明的，所以例子中的类是Calculator
    Class<?> declaringClass = field.getDeclaringClass();
    //例子中的srcType为SubCalculator
    return resolveType(fieldType, srcType, declaringClass);
}
```

接下来我们看`resolveType()`

```java
/**
*可以看到，此方法的作用就是根据参入的类型不同，调用不同的方法进行解析
**/
private static Type resolveType(Type type, Type srcType, Class<?> declaringClass) {
    if (type instanceof TypeVariable) {
      return resolveTypeVar((TypeVariable<?>) type, srcType, declaringClass);
    } else if (type instanceof ParameterizedType) {
      //type为java.util.List<T>，所以会走此判断
      return resolveParameterizedType((ParameterizedType) type, srcType, declaringClass);
    } else if (type instanceof GenericArrayType) {
      return resolveGenericArrayType((GenericArrayType) type, srcType, declaringClass);
    } else {
      //如果都不符合，证明不是泛型类型，则直接返回原类型
      return type;
    }
  }
```

这里我们先看`resolveParameterizedType()`方法。

```java
  private static ParameterizedType resolveParameterizedType(ParameterizedType 				parameterizedType, Type srcType, Class<?> declaringClass) {
    //获取原始类型	例子中为java.util.List
    Class<?> rawType = (Class<?>) parameterizedType.getRawType();
    //获取真实类型，也就是泛型中<>里面的类型，例子中为T，
    //由于可能存在多个类型或者是嵌套关系，如Map<K,V>或List<Map<K,V>>所以这里使用Type数组接收
    Type[] typeArgs = parameterizedType.getActualTypeArguments();
    Type[] args = new Type[typeArgs.length];
    for (int i = 0; i < typeArgs.length; i++) {
      //因为可能存在泛型嵌套的情况，如List<Map<>>，所以这里要递归判断
      if (typeArgs[i] instanceof TypeVariable) {
        //T属于TypeVariable，所以会走此分支，其实这里走哪个分支逻辑都是类似的
        args[i] = resolveTypeVar((TypeVariable<?>) typeArgs[i], srcType, declaringClass);
      } else if (typeArgs[i] instanceof ParameterizedType) {
        args[i] = resolveParameterizedType((ParameterizedType) typeArgs[i], srcType, declaringClass);
      } else if (typeArgs[i] instanceof WildcardType) {
        args[i] = resolveWildcardType((WildcardType) typeArgs[i], srcType, declaringClass);
      } else {
        //如果都不是，证明此时已经不是泛型，直接使用当前类型
        args[i] = typeArgs[i];
      }
    }
    //注意此处的对象是MyBatis自己实现的，位于org.apache.ibatis.reflection.TypeParameterResolver.ParameterizedTypeImpl
    return new ParameterizedTypeImpl(rawType, null, args);
  }
```

下面我们来分析`resolveTypeVar()`方法，注意！！！，此方法和它里面调用的`scanSuperTypes()`是此类的关键核心代码。

```java
  private static Type resolveTypeVar(TypeVariable<?> typeVar, Type srcType, Class<?> declaringClass) {
    Type result = null;
    Class<?> clazz = null;
    //判断srcType是Class类型还是泛型类型，因为例子中的是SubCalculator，所以属于Class类型
    if (srcType instanceof Class) {
      clazz = (Class<?>) srcType;
    } else if (srcType instanceof ParameterizedType) {
      ParameterizedType parameterizedType = (ParameterizedType) srcType;
      //如果是parameterizedType类型，则取出srcType的原始类型，
      clazz = (Class<?>) parameterizedType.getRawType();
    } else {
      //通过前面的解析，这里应该不会再有其它类型了，如果存在则抛出异常
      throw new IllegalArgumentException("The 2nd arg must be Class or ParameterizedType, but was: " + srcType.getClass());
    }

    //如果当前类就是声明变量的类，由于当前类的泛型也只是形参，并不存在实参，所以直接返回Object
    if (clazz == declaringClass) {
      Type[] bounds = typeVar.getBounds();
      if(bounds.length > 0) {
        return bounds[0];
      }
      return Object.class;
    }

    //如果在此类没有找到此变量 则继续寻找它的父类和接口
    //例子中superclass的值为Calculator<java.lang.String>
    //注意此处不是Calculator的Class对象，而是SubCalculator中继承的那个Calculator
    //两者的区别是，继承的Class泛型内已经有了实参，而原Class对象泛型内还是形参
    Type superclass = clazz.getGenericSuperclass();
    //开始扫描父类进行解析
    result = scanSuperTypes(typeVar, srcType, declaringClass, clazz, superclass);
    if (result != null) {
      return result;
    }

    Type[] superInterfaces = clazz.getGenericInterfaces();
    for (Type superInterface : superInterfaces) {
      result = scanSuperTypes(typeVar, srcType, declaringClass, clazz, superInterface);
      if (result != null) {
        return result;
      }
    }
    //如果经过以上步骤还是没有解析出来真实类型，则直接返回Object
    return Object.class;
  }

```

接着我们继续解析`scanSuperTypes()`方法，敲黑板！此方法也是此类的核心代码

```java
  private static Type scanSuperTypes(TypeVariable<?> typeVar, Type srcType, Class<?> 		declaringClass, Class<?> clazz, Type superclass) {
    Type result = null;
    //例子中superclass的值为Calculator<java.lang.String>
    if (superclass instanceof ParameterizedType) {
      //转换为ParameterizedType对象
      ParameterizedType parentAsType = (ParameterizedType) superclass;
      //换取原始类型，例子中为Calculator
      Class<?> parentAsClass = (Class<?>) parentAsType.getRawType();
      //如果此类是声明变量的类
      if (declaringClass == parentAsClass) {
        //取出类的泛型，例子中为java.lang.String
        Type[] typeArgs = parentAsType.getActualTypeArguments();
        //获取声明类的泛型，此处为T
        TypeVariable<?>[] declaredTypeVars = declaringClass.getTypeParameters();
        //循环T
        for (int i = 0; i < declaredTypeVars.length; i++) {
          //查找到字段泛型的形参对应类泛型形参的位置
          if (declaredTypeVars[i] == typeVar) {
            //如果继承的类泛型也是形参
            //由于例子中的为java.lang.String所以这里直接返回
            if (typeArgs[i] instanceof TypeVariable) {
              //取出当前类的泛型
              TypeVariable<?>[] typeParams = clazz.getTypeParameters();
              for (int j = 0; j < typeParams.length; j++) {
				//找到继承类泛型的形参对应的当前类泛型的形参
                if (typeParams[j] == typeArgs[i]) {
                  if (srcType instanceof ParameterizedType) {
                    //根据索引找到泛型对应的实参，
                    result = ((ParameterizedType) srcType).getActualTypeArguments()[j];
                  }
                  break;
                }
              }
            } else {   //如果继承的类泛型不是形参，则直接返回对应的实参类型
              result = typeArgs[i];
            }
          }
        }
          //如果声明字段的类是继承类的父类，则继续向上解析
      } else if (declaringClass.isAssignableFrom(parentAsClass)) {
        result = resolveTypeVar(typeVar, parentAsType, declaringClass);
      }
        //如果继承的类是Class对象，则继续向上解析
    } else if (superclass instanceof Class) {
      if (declaringClass.isAssignableFrom((Class<?>) superclass)) {
        result = resolveTypeVar(typeVar, superclass, declaringClass);
      }
    }
    return result;
  }
```

接下来我们来看解析泛型数组用到的方法`resolveGenericArrayType()`

```java
  private static Type resolveGenericArrayType(GenericArrayType genericArrayType, Type srcType, Class<?> declaringClass) {
    Type componentType = genericArrayType.getGenericComponentType();
    Type resolvedComponentType = null;
    //这里的逻辑与前面类似，不再过多阐述
    if (componentType instanceof TypeVariable) {
      resolvedComponentType = resolveTypeVar((TypeVariable<?>) componentType, srcType, declaringClass);
    } else if (componentType instanceof GenericArrayType) {
      resolvedComponentType = resolveGenericArrayType((GenericArrayType) componentType, srcType, declaringClass);
    } else if (componentType instanceof ParameterizedType) {
      resolvedComponentType = resolveParameterizedType((ParameterizedType) componentType, srcType, declaringClass);
    }
      
    //如果解析到Class对象，则直接返回
    if (resolvedComponentType instanceof Class) {
      return Array.newInstance((Class<?>) resolvedComponentType, 0).getClass();
    } else {
      //否则返回GenericArrayTypeImpl对象
      return new GenericArrayTypeImpl(resolvedComponentType);
    }
  }

```

关于`TypeParameterResolver`类的解析就到这里，其余方法如`resolveWildcardType()`逻辑与上面都类似，这里就不在过多阐述了，有兴趣的读者可以自行查看源码学习。





