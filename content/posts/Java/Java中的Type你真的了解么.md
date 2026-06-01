---
title: Java中的Type你真的了解么
tags:
  - Java
date: 2018-07-13
categories:
 - Java
---
## Interface Type  ##

> Type是Java语言中所有类型的变量，包含基本类型、参数类型、数组类型、类型变量和原始类型

上面是`JavaAPI`中对`Type`接口的描述。注意，这里的**所有类型**并不是我们平时所说的`int`，`String`等这种数据类型，而是基本包含了我们能想象到的**所有类型**（如基本数据类型，集合类型，泛型等），是所有类型的父类，类似于`Object`类是所有类的父类。此接口中只有一个默认方法，通过注释可以发现该方法返回类型的描述信息，而且是从`1.8`版本才加入。
		
```java
default String getTypeName() {
    return toString();
}
```


此接口有四个子接口和一个实现类，除了`Class`外，基本上四个类型都是为泛型来服务的，如下图

![](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/%E5%8D%9A%E5%AE%A2/Type%E7%B1%BB%E5%9B%BE.png)

### Class ###

`Class`是该类的唯一一个实现类，此类比较好理解，也是我们最常见的，一个`Class`对象表示`Java`应用程序的一个类或者接口。枚举和注解属于接口，`Java`中的原始数据类型`int`，`double`还有关键字`void`也都属于`Class`对象，数组也被映射为`Class`对象。  
该类获取比较简单，可以通过`对象名.getClass()`，`类型.class`,`Class.forName()`方式获得，此类也是反射模块中比较重要的类，可以通过获取到的`Class`对象获取字段和方法等。

### ParameterizedType ###

`ParameterizedType`代表一个参数化的类型，比如`Collection<String>`，`List<String>`，该接口中有三个方法

```java
Type[] getActualTypeArguments();

Type getRawType();

Type getOwnerType();
```

#### getActualTypeArguments ####

`getActualTypeArguments()`方法代表泛型的实际类型，如`List<String>`则返回`String`，因为类型可能是多个，比如`Map<String,String>`,所以会返回数组。需要注意的是，该方法只会返回泛型中最外层的类型，考虑到以下情况`List<List<T>>`，也只会返回`List<T>`。

#### getRawType ####

`getRawType()`方法代表该泛型的所属类型，也可以理解为该泛型的原始类型，如`List<String>`则返回`List`，`Collection<String>`则返回`Collection`。

#### getOwnerType ####

`getOwnerType()`代表该泛型原始类型的所属类型。与上个方法有区别的是，如泛型`List<String>`，`getRawType()`方法表示整个`List<String>`泛型的所属类型，而`getOwnerType()`表示`List`的所属类型。
这里所说的所属类型，是指：考虑`A`中有内部类`B`，存在`A.B`，则我们说`B`的所属类型是`A`。此关系最常见的是`Map`和`Entry`的关系，这里的`Map`就是`Entry`的所属类型，如果该类型没有所属类型，则会返回`null`。

#### 示例 ####

```java
public class ParameterizedTypeTest<T> {

    public List<T> list;

    public static void main(String[] args){
        try {
            Class<ParameterizedTypeTest> parameterizedTypeTestClass = 
				ParameterizedTypeTest.class;

            Field list = parameterizedTypeTestClass.getField("list");

            Type type = list.getGenericType();

            System.out.printf("className:\t%s\n", type.getClass().getName());

            ParameterizedType genericType = (ParameterizedType) type;

            Type[] actualTypeArguments = genericType.getActualTypeArguments();

            for (int i = 0; i < actualTypeArguments.length; i++) {
                System.out.printf("actualTypeArguments(%s):\t%s\n", i, 
					actualTypeArguments[i]);
                System.out.printf("actualTypeArgumentsClassName(%s):\t%s\n", i, 
					actualTypeArguments[i].getClass().getName());
            }
            
            Type rawType = genericType.getRawType();

            System.out.printf("getRawType:\t%s\n", rawType);
            System.out.printf("getRawTypeClassName:\t%s\n", 
				rawType.getClass().getName());

            Type ownerType = genericType.getOwnerType();

            System.out.printf("getOwnerType:\t%s\n", ownerType);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```


​	

以上代码运行结果为

```java
className:	sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
actualTypeArguments(0):	T
actualTypeArgumentsClassName(0):
		sun.reflect.generics.reflectiveObjects.TypeVariableImpl
getRawType:	interface java.util.List
getRawTypeClassName:	java.lang.Class
getOwnerType:	null		//这里由于List是顶级接口，所以返回null
```


根据结果可以看到，`actualTypeArgumentsClassName(0)`的结果为`sun.reflect.generics.reflectiveObjects.TypeVariableImpl`，下面我们就一起认识一下`TypeVariable`。

### TypeVariable ###

`TypeVariable`代表一个类型变量，它可以显式的被类型，方法和构造函数的类型声明，也可以隐式的被声明。  

前面说的可能有些绕，简单来说，`TypeVariable表`示就是泛型中的具体类型，如`List<String>`,代表的就是`String`，`List<T>`代表的就是T。此接口的方法有

```java
Type[] getBounds();

D getGenericDeclaration();

String getName();

AnnotatedType[] getAnnotatedBounds();
```

#### getBounds ####
`getBounds()`代表泛型中实际类型的上界，考虑到以下情况`List<T extends String>`，泛型中`T`的上界是`String`，那么该方法就会返回`String`,值得注意的是，如果没有`extends`关键字，则会默认继承`Object`类

#### getGenericDeclaration ####

`getGenericDeclaration()`代表泛型的原始类型，如`List<T extends String>`，则该方法返回`List`

#### getName ####

`getName()`代表泛型中实际类型的名称，如`List<T extends String>`，则该方法返回`T`

#### getAnnotatedBounds ####

`getAnnotatedBounds()`代表泛型中实际类型的上界，考虑到以下情况`List<T extends String>`，泛型中`T`的上界是`String`，那么该方法就会返回`String`，此方法类似于`getBounds()`，不过返回值为`AnnotatedType[]`。

{% blockquote Java, API %}
`AnnotatedType`表示当前在此`JVM`中运行的程序中类型的潜在注释用法。 该用途可以是`Java`编程语言中的任何类型，包括数组类型，参数化类型，类型变量或通配符类型。
{% endblockquote %}


#### 示例 ####

```java
public class TypeVariableTest<T extends String> {

    public List<T> list;

    public static void main(String[] args){
        try {
            Class<TypeVariableTest> typeVariableTestClass = TypeVariableTest.class;

            Field list = typeVariableTestClass.getField("list");

            ParameterizedType genericType = (ParameterizedType) list.getGenericType();

            Type[] actualTypeArguments = genericType.getActualTypeArguments();

            Type type = actualTypeArguments[0];

            System.out.printf("actualTypeArgumentsClassName:\t%s\n", 
				type.getClass().getName());

            TypeVariable typeVariable = (TypeVariable) type;

            Type[] bounds = typeVariable.getBounds();

            for (int i = 0; i < bounds.length; i++) {
                System.out.printf("getBounds:\t%s\n", bounds[i]);
            }

            GenericDeclaration genericDeclaration = 
				typeVariable.getGenericDeclaration();

            System.out.printf("getGenericDeclaration:\t%s\n", genericDeclaration);

            String name = typeVariable.getName();

            System.out.printf("name:\t%s\n", name);

            AnnotatedType[] annotatedBounds = typeVariable.getAnnotatedBounds();

            for (int i = 0; i < annotatedBounds.length; i++) {
                System.out.printf("annotatedBounds:\t%s\n", 
					annotatedBounds[i].getType());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```


​	

   


​	



以上代码运行结果为

```java
actualTypeArgumentsClassName:	sun.reflect.generics.reflectiveObjects.TypeVariableImpl
getBounds:	class java.lang.String
getGenericDeclaration:	class reflector.TypeVariableTest
name:	T
annotatedBounds:	class java.lang.String
```

### GenericArrayType ###

`GenericArrayType`代表数组类型，如`List<String>[]`，此接口只有一个方法

```java
 Type getGenericComponentType();
```

#### getGenericComponentType ####
`getGenericComponentType()`返回真实类型，如`List<String>[]`，则返回`List<String>`。需要注意的是，如果有多维数组，也只会去掉单个维度，如`List<String>[][][][]`，最后的返回结果是`List<String>[][][]`。

#### 示例 ####

```java
public class GenericArrayTypeTest<T> {

    public List<T>[] lists;

    public static void main(String[] args) {
        try {
            Class<GenericArrayTypeTest> genericArrayTypeTestClass = GenericArrayTypeTest.class;

            Field lists = genericArrayTypeTestClass.getField("lists");

            Type type = lists.getGenericType();

            System.out.printf("className:\t%s\n", type.getClass().getName());

            GenericArrayType genericType = (GenericArrayType) type;

            Type genericComponentType = genericType.getGenericComponentType();

            System.out.printf("genericComponentType:\t%s\n", genericComponentType);
            System.out.printf("genericComponentTypeClassName:\t%s\n", 
				genericComponentType.getClass().getName());

        } catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

以上代码运行结果为

```java
className:	sun.reflect.generics.reflectiveObjects.GenericArrayTypeImpl
genericComponentType:	java.util.List<T>
genericComponentTypeClassName:
	sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl
```

注意最后一个结果`genericComponentTypeClassName:	sun.reflect.generics.reflectiveObjects.ParameterizedTypeImpl`，因为此处返回的是`List<T>`，所以`Class`类型为`ParameterizedTypeImpl`，如果字段类型是`List<T>[][]`，那么返回的就是`List<T>[]`，这里的`Class`类型就是`GenericArrayType`。

### WildcardType ###

`WildcardType` 代表一个通配符类型的表达式, 比如 `?`, `? extends Number`, 或者 `? super Integer`。该接口有两个方法

```java
Type[] extendsBounds();

Type[] superBounds();
```

#### extendsBounds ####

`extendsBounds()` 该方法代表泛型的上界，如`? extends Number`，则返回`Number`，需要注意的是，如果不显示的声明`extends`，那么泛型都会隐式的继承`Object`。

#### superBounds ####

`superBounds()` 该方法代表泛型的下界，如`? super Integer`，则返回`Integer`。

#### 示例 ####

首先看一下`extends`的情况

```java
public class WildcardTypeTest {

    public List<? extends String> list;

    public static void main(String[] args){
        try {
            Class<WildcardTypeTest> wildcardTypeTestClass = WildcardTypeTest.class;

            Field list = wildcardTypeTestClass.getField("list");

            ParameterizedType genericType = (ParameterizedType) list.getGenericType();

            Type[] actualTypeArguments = genericType.getActualTypeArguments();

            Type type = actualTypeArguments[0];

            System.out.printf("actualTypeArgumentsClassName:\t%s\n", type);
            System.out.printf("actualTypeArgumentsClassName:\t%s\n", type.getClass().getName());

            WildcardType wildcardType = (WildcardType) type;

            Type[] lowerBounds = wildcardType.getLowerBounds();

            System.out.printf("lowerBounds_size:\t%s\n", lowerBounds.length);

            for (int i = 0; i < lowerBounds.length; i++) {
                System.out.printf("lowerBounds:\t%s\n", lowerBounds[i]);
                System.out.printf("lowerBounds:\t%s\n", lowerBounds[i].getClass().getName());

            }

            Type[] upperBounds = wildcardType.getUpperBounds();

            System.out.printf("upperBounds_size:\t%s\n", upperBounds.length);

            for (int i = 0; i < upperBounds.length; i++) {
                System.out.printf("upperBounds:\t%s\n", upperBounds[i]);
                System.out.printf("upperBounds:\t%s\n", upperBounds[i].getClass().getName());

            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```


​	

输出结果为

```java
actualTypeArgumentsClassName:	? extends java.lang.String
actualTypeArgumentsClassName:	sun.reflect.generics.reflectiveObjects.WildcardTypeImpl
lowerBounds_size:	0
upperBounds_size:	1
upperBounds:	class java.lang.String
upperBounds:	java.lang.Class
```

可以看到，当关键字为`extends`的时候，该类型的下界为空，再来看一下`super`的场景

```java
public class WildcardTypeTest {

    public List<? super String> list;

    public static void main(String[] args){
        try {
            Class<WildcardTypeTest> wildcardTypeTestClass = WildcardTypeTest.class;

            Field list = wildcardTypeTestClass.getField("list");

            ParameterizedType genericType = (ParameterizedType) list.getGenericType();

            Type[] actualTypeArguments = genericType.getActualTypeArguments();

            Type type = actualTypeArguments[0];

            System.out.printf("actualTypeArgumentsClassName:\t%s\n", type);
            System.out.printf("actualTypeArgumentsClassName:\t%s\n", type.getClass().getName());

            WildcardType wildcardType = (WildcardType) type;

            Type[] lowerBounds = wildcardType.getLowerBounds();

            System.out.printf("lowerBounds_size:\t%s\n", lowerBounds.length);

            for (int i = 0; i < lowerBounds.length; i++) {
                System.out.printf("lowerBounds:\t%s\n", lowerBounds[i]);
                System.out.printf("lowerBounds:\t%s\n", lowerBounds[i].getClass().getName());

            }

            Type[] upperBounds = wildcardType.getUpperBounds();

            System.out.printf("upperBounds_size:\t%s\n", upperBounds.length);

            for (int i = 0; i < upperBounds.length; i++) {
                System.out.printf("upperBounds:\t%s\n", upperBounds[i]);
                System.out.printf("upperBounds:\t%s\n", upperBounds[i].getClass().getName());

            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

输出结果为

```java
actualTypeArgumentsClassName:	? super java.lang.String
actualTypeArgumentsClassName:	sun.reflect.generics.reflectiveObjects.WildcardTypeImpl
lowerBounds_size:	1
lowerBounds:	class java.lang.String
lowerBounds:	java.lang.Class
upperBounds_size:	1
upperBounds:	class java.lang.Object
upperBounds:	java.lang.Class
```

可以看到，由于没有声明`extends`，这里的类型上界为`java.lang.Object`。

