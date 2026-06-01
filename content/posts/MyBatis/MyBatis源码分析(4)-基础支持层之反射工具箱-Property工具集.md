---
title: MyBatis源码分析(4)-基础支持层之反射工具箱-Property工具集
tags:
  - MyBatis
date: 2018-08-05
categories:
 - MyBatis
---



## Property工具集

本篇内容我们主要介绍`MyBatis`中的`Property`工具集，该工具集的作用主要是解析表达式或提取属性名称，主要介绍三个类，分别是`PropertyCopier`、`PropertyNamer`和`PropertyTokenizer`。由于这三个类比较简单，所以放在一篇文章中介绍。

### PropertyCopier

首先来介绍`PropertyCopier`，根据类名可以看出，该类主要用途为属性复制，该类只有一个方法`copyBeanProperties()`，我们先来测试一下此方法。

```java
public class Calculator<T> {
  protected T id;

  public T getId() {
    return id;
  }

  public void setId(T id) {
    this.id = id;
  }

  public static class SubCalculator extends Calculator<String> {
    public String name;
  }
}
```

```java
Calculator.SubCalculator subCalculator1 = new Calculator.SubCalculator();

subCalculator1.name = "subCalculator1";
subCalculator1.setId("subCalculator1-Id");

System.out.printf("subCalculator1.name:%s\n",subCalculator1.name);
System.out.printf("subCalculator1.id:%s\n",subCalculator1.getId());

Calculator.SubCalculator subCalculator2 = new Calculator.SubCalculator();

System.out.println("复制之前-----");
System.out.printf("subCalculator2.name:%s\n",subCalculator2.name);
System.out.printf("subCalculator2.id:%s\n",subCalculator2.getId());

//该方法主要参数为：#1源对象的所属Class，#2源对象，#3目标对象
PropertyCopier.copyBeanProperties(subCalculator1.getClass(), subCalculator1, subCalculator2);

System.out.println("复制之后-----");
System.out.printf("subCalculator2.name:%s\n",subCalculator2.name);
System.out.printf("subCalculator2.id:%s\n",subCalculator2.getId());
```

以上代码运行结果为

```java
subCalculator1.name:subCalculator1
subCalculator1.id:subCalculator1-Id
复制之前-----
subCalculator2.name:null
subCalculator2.id:null
复制之后-----
subCalculator2.name:subCalculator1
subCalculator2.id:subCalculator1-Id
```

接下来我们看一下该方法的主要实现过程

```java
public static void copyBeanProperties(Class<?> type, Object sourceBean, Object destinationBean) {
    Class<?> parent = type;
    //因为源对象可能会存在父类，循环赋值
    while (parent != null) {
      //获取该对象所属Class的所有字段
      final Field[] fields = parent.getDeclaredFields();
      for(Field field : fields) {
        try {
          field.setAccessible(true);
          //取出源对象的该字段，为目标对象的该字段赋值
          field.set(destinationBean, field.get(sourceBean));
        } catch (Exception e) {
          //因为这里只会在为final字段赋值时发生异常，所以不需要处理
        }
      }
      //继续查找父类
      parent = parent.getSuperclass();
    }
  }
```

可以看到，方法实现比较简单，主要依靠`field.set()`和`field.get()`方法实现。这里就不再过多阐述。

### PropertyNamer

该类主要的作用是对方法名的一些处理，主要有四个方法：`methodToProperty()`：从方法名中取出属性名，该方法我们之前在分析`Reflector`的时候有用到过。`isProperty()`：判断给定方法名是不是属性，`isGetter()\isSetter()`：判断是否是getter或者setter方法。

按照惯例，先测试一下这几个方法。

```java
//首先，模拟方法名为getName、setName
String getName = "getName";
String setName = "setName";
System.out.printf("methodToProperty-getName：%s\n",PropertyNamer.methodToProperty(getName));
System.out.printf("methodToProperty-setName：%s\n",PropertyNamer.methodToProperty(setName));
System.out.printf("isGetter-getName：%s\n",PropertyNamer.isGetter(getName));
System.out.printf("isSetter-getName：%s\n",PropertyNamer.isSetter(getName));
System.out.printf("isGetter-setName：%s\n",PropertyNamer.isGetter(setName));
System.out.printf("isSetter-setName：%s\n",PropertyNamer.isSetter(setName));
System.out.printf("isProperty-getName：%s\n",PropertyNamer.isProperty(getName));
System.out.printf("isProperty-setName：%s\n",PropertyNamer.isProperty(setName));
```

输出结果为

```java
methodToProperty-getName：name
methodToProperty-setName：name
isGetter-getName：true
isSetter-getName：false
isGetter-setName：false
isSetter-setName：true
isProperty-getName：true
isProperty-setName：true
```

测试后，一起来看看方法的实现，由于比较简单，所以这里就一起介绍了

```java
private PropertyNamer() {
    //由于方法都是static的，所以不允许实例化
  }

  public static String methodToProperty(String name) {
    //如果方法名以is开头，则从第二位开始截取
    if (name.startsWith("is")) {
      name = name.substring(2);
    } else if (name.startsWith("get") || name.startsWith("set")) {
      //如果方法名以get或者set开头，则从第三位开始截取
      name = name.substring(3);
    } else {
      //如果都不是则抛出异常
      throw new ReflectionException("Error parsing property name '" + name + "'.  Didn't start with 'is', 'get' or 'set'.");
    }

    //由于方法名都是驼峰命名，如setName，所以要把截取出来的名称第一位转换为小写
    //如果名称长度是1，或者大于1并且第一位是大写
    if (name.length() == 1 || (name.length() > 1 && !Character.isUpperCase(name.charAt(1)))) {
      //将第一位转换为小写
      name = name.substring(0, 1).toLowerCase(Locale.ENGLISH) + name.substring(1);
    }

    return name;
  }

  //判断是否是属性
  public static boolean isProperty(String name) {
    return name.startsWith("get") || name.startsWith("set") || name.startsWith("is");
  }

  //判断是否是getter
  public static boolean isGetter(String name) {
    return name.startsWith("get") || name.startsWith("is");
  }
  //判断是否是setter
  public static boolean isSetter(String name) {
    return name.startsWith("set");
  }
```

### PropertyTokenizer

考虑以下场景：一个用户对应多个角色，一个角色对应多个权限，我们想查出第一个角色对应的前两个权限名称，假设数据格式如下（此处只是举例，真实场景可能不会这样设计）



| user |  role  | perm | perm2 |
| :--: | :----: | :--: | :---: |
| 张三 | 管理员 | 删除 | 查看  |



那么我们的实体类可能会这样设计

```java
public class User {
    private String userName;
    private List<Role> roles;  
}

/**
 * 角色
 * **/
public class Role {
    private String roleName;
    private List<Perm> perms;
}

/**
 * 权限
 * **/
public class Perm {
    private String permName;
}
```

那么我们在配置映射关系的时候，就会如下配置

```xml
<resultMap id="test" type="User" >
<id property="userName" column="user" />
<result property= "roles[0].perms[0].permName" column= "perm" />
<result property= "roles[0].perms[1].permName" column= "perm" />
</resultMap>
```

类似于上方中的`roles[0].perms[0].permName`这种复杂的表达式，就是由`PropertyTokenizer`来解析的。按照惯例，先运行，后介绍。

```java
String fullName = "roles[0].perms[1].permName";

PropertyTokenizer propertyTokenizer = new PropertyTokenizer(fullName);

System.out.printf("name:%s\n",propertyTokenizer.getName());

System.out.printf("index:%s\n",propertyTokenizer.getIndex());

System.out.printf("IndexedName:%s\n",propertyTokenizer.getIndexedName());

while (propertyTokenizer.hasNext()) {

    System.out.println("-----next-----");

    propertyTokenizer = propertyTokenizer.next();

    System.out.printf("name:%s\n",propertyTokenizer.getName());

    System.out.printf("index:%s\n",propertyTokenizer.getIndex());

    System.out.printf("IndexedName:%s\n",propertyTokenizer.getIndexedName());

}
```

输出结果为

```java
name:roles
index:0
IndexedName:roles[0]
-----next-----
name:perms
index:1
IndexedName:perms[1]
-----next-----
name:permName
index:null
IndexedName:permName
```

根据输出结果可以看出，`PropertyTokenizer`以`.`为基准将一个表达式分为多段，每段为一个对象，类似于链表结构，通过`next()`方法指向下一个对象。`name`中存储的是名称（不包含索引值）,`index`表示索引，`indexedName`表示带索引的名称。下面我们一起看一下代码实现

```java
public class PropertyTokenizer implements Iterator<PropertyTokenizer> {
  //前面三个字段之前已经说过，这里不再阐述
  private String name;
  private String indexedName;
  private String index;
  //保存子集,也就是第一个“.”后的内容
  private String children;

  public PropertyTokenizer(String fullname) {
    //为了便于理解，我们拿测试中的值fullName = "roles[0].perms[1].permName"来分析
    //查找第一个.所在位置
    int delim = fullname.indexOf('.');
    //如果找到了
    if (delim > -1) {
      //截取小数点之前的值  这里是roles[0]
      name = fullname.substring(0, delim);
      //小数点之后的值为子集，这里是perms[1].permName
      children = fullname.substring(delim + 1);
    } else {
      //如没有找到，则name为当前值，并且没有子集
      name = fullname;
      children = null;
    }
    
    indexedName = name;
    
    //开始截取索引
    delim = name.indexOf('[');
    if (delim > -1) {
      //保存index和name
      //这里的index为0  name为roles
      index = name.substring(delim + 1, name.length() - 1);
      name = name.substring(0, delim);
    }
  }

  //get set方法省略
  
  @Override
  public boolean hasNext() {
    //如果子集不为空，就是还有下一个
    return children != null;
  }
    
  @Override
  public PropertyTokenizer next() {
    //继续解析余下内容
    return new PropertyTokenizer(children);
  }

  @Override
  public void remove() {
    //不支持remove
    throw new UnsupportedOperationException("Remove is not supported, as it has no meaning in the context of properties.");
  }
}
```

可以看到，该类继承了`Iterator`迭代器，实现了`hasNext（）`和`next()`方法。需要注意的是，该类并不支持`remove()`方法。