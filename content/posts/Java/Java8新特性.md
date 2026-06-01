---
title: Java8新特性
tags:
  - Java
date: 2018-03-01
categories:
 - Java
---
## 声明

本文部分内容摘抄自[菜鸟教程][1]，仅作学习记录之用。

Java 8 新特性
--

`Java 8` (又称为` jdk 1.8`) 是` Java `语言开发的一个主要版本。 `Oracle `公司于 `2014 年 3 月 18 日`发布` Java 8` ，它支持函数式编程，新的` JavaScript `引擎，新的日期 `API`，新的`Stream API `等。`Java8 `新增了非常多的特性，本文只讨论下面几点。

 - `Lambda` 表达式 − `Lambda`允许把函数作为一个方法的参数（函数作为参数传递进方法中）。
 - 方法引用 − 方法引用提供了非常有用的语法，可以直接引用已有Java类或对象（实例）的方法或构造器。与lambda联合使用，方法引用可以使语言的构造更紧凑简洁，减少冗余代码。
 - 默认方法 − 默认方法就是一个在接口里面有了一个实现的方法。实现类非必须实现此方法。
 - `Stream API `−新添加的`Stream API（java.util.stream） `把真正的函数式编程风格引入到`Java`中。可以把`Collection`转换为流操作，大大提升了操作效率和代码简洁度。
 - `Date Time API `− 加强对日期与时间的处理。
 - `Optional `类 − `Optional `类已经成为 `Java 8 `类库的一部分，用来解决空指针异常。




Lambda 表达式
--
`Lambda `表达式，也可称为闭包，它是推动` Java 8 `发布的最重要新特性。

`Lambda` 允许把函数作为一个方法的参数（函数作为参数传递进方法中）。

使用` Lambda` 表达式可以使代码变的更加简洁紧凑。
### 语法 ###
`lambda `表达式的语法格式如下：

    (parameters) -> expression
或

    (parameters) ->{ statements; }
以下是`lambda`表达式的重要特征:

 - 可选类型声明：不需要声明参数类型，编译器可以统一识别参数值。
 - 可选的参数圆括号：一个参数无需定义圆括号，但多个参数需要定义圆括号。
 - 可选的大括号：如果主体包含了一个语句，就不需要使用大括号。
 - 可选的返回关键字：如果主体只有一个表达式返回值则编译器会自动返回值，大括号需要指明表达式返回了一个数值。
### Lambda 表达式实例 ###
```java
public class Java8Tester {
public static void main(String args[]){
  Java8Tester tester = new Java8Tester();
      // 类型声明
  MathOperation addition = (int a, int b) -> a + b;
    
  // 不用类型声明
  MathOperation subtraction = (a, b) -> a - b;
    
  // 大括号中的返回语句
  MathOperation multiplication = (int a, int b) -> { return a * b; };
    
  // 没有大括号及返回语句
  MathOperation division = (int a, int b) -> a / b;
    
  System.out.println("10 + 5 = " + tester.operate(10, 5, addition));
  System.out.println("10 - 5 = " + tester.operate(10, 5, subtraction));
  System.out.println("10 x 5 = " + tester.operate(10, 5, multiplication));
  System.out.println("10 / 5 = " + tester.operate(10, 5, division));
    
  // 不用括号
  GreetingService greetService1 = message ->
  System.out.println("Hello " + message);
    
  // 用括号
  GreetingService greetService2 = (message) ->
  System.out.println("Hello " + message);
    
  greetService1.sayMessage("Runoob");
  greetService2.sayMessage("Google");
  }
    
  interface MathOperation {
     int operation(int a, int b);
  }
    
  interface GreetingService {
     void sayMessage(String message);
  }
    
  private int operate(int a, int b, MathOperation mathOperation){
     return mathOperation.operation(a, b);
  }
}  

```


执行以上脚本，输出结果为：

```java
10 + 5 = 15
10 - 5 = 5
10 x 5 = 50
10 / 5 = 2
Hello Runoob
Hello Google
```
使用` Lambda `表达式需要注意以下两点：

 - `Lambda `表达式主要用来定义行内执行的方法类型接口，例如，一个简单方法接口。在上面例子中，我们使用各种类型的`Lambda`表达式来定义`MathOperation`接口的方法。然后我们定义了`sayMessage`的执行。
 - `Lambda `表达式免去了使用匿名方法的麻烦，并且给予Java简单但是强大的函数化的编程能力。
### 变量作用域 ###
`lambda` 表达式只能引用标记了 final 的外层局部变量，这就是说不能在` lambda` 内部修改定义在域外的局部变量，否则会编译错误。

在` Java8Tester.java `文件输入以下代码：

```java
 public class Java8Tester {
 
   final static String salutation = "Hello! ";
   
   public static void main(String args[]){
      GreetingService greetService1 = message -> 
      System.out.println(salutation + message);
      greetService1.sayMessage("Runoob");
   }
    
   interface GreetingService {
      void sayMessage(String message);
   }
}
```

执行以上脚本，输出结果为：

```java
Hello! Runoob
```
我们也可以直接在` lambda `表达式中访问外层的局部变量：

`Java8Tester.java `文件

```java
public class Java8Tester {
    public static void main(String args[]) {
        final int num = 1;
        Converter<Integer, String> s = (param) -> System.out.println(String.valueOf(param + num));
        s.convert(2);  // 输出结果为 3
    }
 
    public interface Converter<T1, T2> {
        void convert(int i);
    }
}
```

`lambda` 表达式的局部变量可以不用声明为 `final`，但是必须不可被后面的代码修改（即隐性的具有 `final `的语义）

```java
int num = 1;  
Converter<Integer, String> s = (param) -> System.out.println(String.valueOf(param + num));
s.convert(2);
num = 5;  
//报错信息：Local variable num defined in an enclosing scope must be final or effectively final
```

在 Lambda 表达式当中不允许声明一个与局部变量同名的参数或者局部变量。

```java
String first = "";  
Comparator<String> comparator = (first, second) -> Integer.compare(first.length(), second.length());  //编译会出错 
```
## 方法引用 ## 

方法引用通过方法的名字来指向一个方法。

方法引用可以使语言的构造更紧凑简洁，减少冗余代码。

方法引用使用一对冒号 :: 。

下面，我们在 Car 类中定义了 4 个方法作为例子来区分 Java 中 4 种不同方法的引用。

```java
@FunctionalInterface
//FunctionalInterface用来声明此接口为函数式接口
public interface Supplier<T> {
    T get();
}
 
class Car {
    //Supplier是jdk1.8的接口，这里和lamda一起使用了
    public static Car create(final Supplier<Car> supplier) {
        return supplier.get();
    }
 
    public static void collide(final Car car) {
        System.out.println("Collided " + car.toString());
    }
 
    public void follow(final Car another) {
        System.out.println("Following the " + another.toString());
    }
 
    public void repair() {
        System.out.println("Repaired " + this.toString());
    }
}
```

 - 构造器引用：它的语法是`Class::new`，或者更一般的`Class< T >::new`实例如下：
    ` final Car car = Car.create( Car::new );
    final List< Car > cars = Arrays.asList( car );`
 - 静态方法引用：它的语法是Class::static_method，实例如下：
    `cars.forEach( Car::collide );`
 - 特定类的任意对象的方法引用：它的语法是Class::method实例如下：
    `cars.forEach( Car::repair );`
 - 特定对象的方法引用：它的语法是instance::method实例如下：
    `final Car police = Car.create( Car::new );
    cars.forEach( police::follow );`



### 方法引用实例 ###
在` Java8Tester.java `文件输入以下代码：

```java
import java.util.List;
import java.util.ArrayList;
 
public class Java8Tester {
   public static void main(String args[]){
      List names = new ArrayList();
        
      names.add("Google");
      names.add("Runoob");
      names.add("Taobao");
      names.add("Baidu");
      names.add("Sina");
        
      names.forEach(System.out::println);
   }
}
```
实例中我们将 `System.out::println `方法作为静态方法来引用。

执行以上脚本，输出结果为：

    Google
    Runoob
    Taobao
    Baidu
    Sina

函数式接口
--
函数式接口`(Functional Interface)`就是一个具有一个方法的普通接口。

函数式接口可以被隐式转换为`lambda`表达式。

函数式接口可以现有的函数友好地支持` lambda`。

JDK 1.8之前已有的函数式接口:

 - `java.lang.Runnable`

 - `java.util.concurrent.Callable`

 - `java.security.PrivilegedAction`

 - `java.util.Comparator`

 - `java.io.FileFilter`

 - `java.nio.file.PathMatcher`

 - `java.lang.reflect.InvocationHandler`

 - `java.beans.PropertyChangeListener`

 - `java.awt.event.ActionListener`

 - `javax.swing.event.ChangeListener`

  `JDK 1.8` 新增加的函数接口：

 - `java.util.function`
### 函数式接口实例 ###
`Predicate <T> `接口是一个函数式接口，它接受一个输入参数` T`，返回一个布尔值结果。

该接口包含多种默认方法来将Predicate组合成其他复杂的逻辑（比如：与，或，非）。

该接口用于测试对象是` true `或` false`。

我们可以通过以下实例（`Java8Tester.java`）来了解函数式接口` Predicate <T> `的使用：

```java
import java.util.Arrays;
import java.util.List;
import java.util.function.Predicate;
 
public class Java8Tester {
   public static void main(String args[]){
      List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9);
        
      // Predicate<Integer> predicate = n -> true
      // n 是一个参数传递到 Predicate 接口的 test 方法
      // n 如果存在则 test 方法返回 true
        
      System.out.println("输出所有数据:");
        
      // 传递参数 n
      eval(list, n->true);
        
      // Predicate<Integer> predicate1 = n -> n%2 == 0
      // n 是一个参数传递到 Predicate 接口的 test 方法
      // 如果 n%2 为 0 test 方法返回 true
        
      System.out.println("输出所有偶数:");
      eval(list, n-> n%2 == 0 );
        
      // Predicate<Integer> predicate2 = n -> n > 3
      // n 是一个参数传递到 Predicate 接口的 test 方法
      // 如果 n 大于 3 test 方法返回 true
        
      System.out.println("输出大于 3 的所有数字:");
      eval(list, n-> n > 3 );
   }
    
   public static void eval(List<Integer> list, Predicate<Integer> predicate) {
      for(Integer n: list) {
        
         if(predicate.test(n)) {
            System.out.println(n + " ");
         }
      }
   }
}
```
执行以上脚本，输出结果为：

```java
输出所有数据:
1 
2 
3 
4 
5 
6 
7 
8 
9 
输出所有偶数:
2 
4 
6 
8 
输出大于 3 的所有数字:
4 
5 
6 
7 
8 
9 
```

## 默认方法
`Java 8 `新增了接口的默认方法。

简单说，默认方法就是接口可以有实现方法，而且不需要实现类去实现其方法。

我们只需在方法名前面加个`default`关键字即可实现默认方法。

为什么要有这个特性？

首先，之前的接口是个双刃剑，好处是面向抽象而不是面向具体编程，缺陷是，当需要修改接口时候，需要修改全部实现该接口的类，目前的`java 8`之前的集合框架没有`foreach`方法，通常能想到的解决办法是在`JDK`里给相关的接口添加新的方法及实现。然而，对于已经发布的版本，是没法在给接口添加新方法的同时不影响已有的实现。所以引进的默认方法。他们的目的是为了解决接口的修改与现有的实现不兼容的问题。
### 语法 ###
默认方法语法格式如下：

```java
public interface vehicle {
   default void print(){
      System.out.println("我是一辆车!");
   }
}
```



### 多个默认方法

一个接口有默认方法，考虑这样的情况，一个类实现了多个接口，且这些接口有相同的默认方法，以下实例说明了这种情况的解决方法：

```java
public interface vehicle {
   default void print(){
      System.out.println("我是一辆车!");
   }
}
 
public interface fourWheeler {
   default void print(){
      System.out.println("我是一辆四轮车!");
   }
}
```

第一个解决方案是创建自己的默认方法，来覆盖重写接口的默认方法：

```java
public class car implements vehicle, fourWheeler {
   default void print(){
      System.out.println("我是一辆四轮汽车!");
   }
}
```

第二种解决方案可以使用 super 来调用指定接口的默认方法：

```java
public class car implements vehicle, fourWheeler {
   public void print(){
      vehicle.super.print();
   }
}
```

### 静态默认方法

Java 8 的另一个特性是接口可以声明（并且可以提供实现）静态方法。例如：

```java
public interface vehicle {
   default void print(){
      System.out.println("我是一辆车!");
   }
    // 静态方法
   static void blowHorn(){
      System.out.println("按喇叭!!!");
   }
}
```

默认方法实例
我们可以通过以下代码来了解关于默认方法的使用，可以将代码放入 Java8Tester.java 文件中：

Java8Tester.java 文件

```java
public class Java8Tester {
   public static void main(String args[]){
      Vehicle vehicle = new Car();
      vehicle.print();
   }
}
 
interface Vehicle {
   default void print(){
      System.out.println("我是一辆车!");
   }
    
   static void blowHorn(){
      System.out.println("按喇叭!!!");
   }
}
 
interface FourWheeler {
   default void print(){
      System.out.println("我是一辆四轮车!");
   }
}
 
class Car implements Vehicle, FourWheeler {
   public void print(){
      Vehicle.super.print();
      FourWheeler.super.print();
      Vehicle.blowHorn();
      System.out.println("我是一辆汽车!");
   }
}
```

执行以上脚本，输出结果为：

```java
我是一辆车!
我是一辆四轮车!
按喇叭!!!
我是一辆汽车!
```

Java 8 Stream
-------------

`Java 8 API`添加了一个新的抽象称为流`Stream`，可以让你以一种声明的方式处理数据。

`Stream `使用一种类似用` SQL `语句从数据库查询数据的直观方式来提供一种对` Java `集合运算和表达的高阶抽象。

`Stream API`可以极大提高`Java`程序员的生产力，让程序员写出高效率、干净、简洁的代码。

这种风格将要处理的元素集合看作一种流， 流在管道中传输， 并且可以在管道的节点上进行处理， 比如筛选， 排序，聚合等。

元素流在管道中经过中间操作`（intermediate operation）`的处理，最后由最终操作`(terminal operation)`得到前面处理的结果。

    +--------------------+       +------+   +------+   +---+   +-------+
    | stream of elements +-----> |filter+-> |sorted+-> |map+-> |collect|
    +--------------------+       +------+   +------+   +---+   +-------+
以上的流程转换为 Java 代码为：

```java
List<Integer> transactionsIds = 
widgets.stream()
             .filter(b -> b.getColor() == RED)
             .sorted((x,y) -> x.getWeight() - y.getWeight())
             .mapToInt(Widget::getWeight)
             .sum();
```

### 什么是 Stream？

`Stream（`流）是一个来自数据源的元素队列并支持聚合操作

元素是特定类型的对象，形成一个队列。` Java`中的`Stream`并不会存储元素，而是按需计算。
数据源 流的来源。 可以是集合，数组，`I/O channel`， 产生器`generator `等。
聚合操作 类似`SQL`语句一样的操作， 比如`filter, map, reduce, find, match, sorted`等。
和以前的`Collection`操作不同，` Stream`操作还有两个基础的特征：

`Pipelining`: 中间操作都会返回流对象本身。 这样多个操作可以串联成一个管道， 如同流式风格`（fluent style）`。 这样做可以对操作进行优化， 比如延迟执行`(laziness)`和短路`( short-circuiting)`。
内部迭代： 以前对集合遍历都是通过`Iterator`或者`For-Each`的方式, 显式的在集合外部进行迭代， 这叫做外部迭代。 `Stream`提供了内部迭代的方式， 通过访问者模式`(Visitor)`实现。

### 示例:


```java
int sum = widgets.stream()
.filter(w -> w.getColor() == RED)
 .mapToInt(w -> w.getWeight())
 .sum();
```
`filter`筛选所有颜色为红色，`mapToInt`映射到int类型，`sum`为结束操作，计算总和。

### Stream流与集合之间的转换:
将集合转换为流:

```java
// 1. 
Stream stream = Stream.of("a", "b", "c");
// 2. 
String [] strArray = new String[] {"a", "b", "c"};
stream = Stream.of(strArray);
stream = Arrays.stream(strArray);
// 3. 
List<String> list = Arrays.asList(strArray);
stream = list.stream();
```
将流转换为其它数据，这时候我们要用到`collect`，`API`中关于`collect`的用法有两种:

```java
//Performs a mutable reduction operation on the elements of this stream using a Collector.
collect(Collector<? super T,A,R> collector)
//Performs a mutable reduction operation on the elements of this stream.
collect(Supplier<R> supplier, BiConsumer<R,? super T> accumulator, BiConsumer<R,R> combiner)
```
配合`collect`使用`Collector`将流转换为集合:

```java
//TreeSet
TreeSet<Integer> collect2 = Stream.of(1, 3, 4).collect(Collectors.toCollection(TreeSet::new));
//list
List<String> list1 = Stream.of(1, 3, 4).collect(Collectors.toList());
//array
String[] strArray1 = Stream.of(1, 3, 4).toArray(String[]::new);
//String
String str = Stream.of(1, 3, 4).collect(Collectors.joining()).toString();
```
由于内容较多，这里只做简单介绍，更多请查看[JAVA API][2]。
Date Time API
--
`Java 8`通过发布新的`Date-Time API (JSR 310)`来进一步加强对日期与时间的处理。

在旧版的` Java `中，日期时间 `API` 存在诸多问题，其中有：

 - 非线程安全 − `java.util.Date `是非线程安全的，所有的日期类都是可变的，这是`Java`日期类最大的问题之一。

 - 设计很差 −` Java`的日期/时间类的定义并不一致，在`java.util`和`java.sql`的包中都有日期类，此外用于格式化和解析的类在`java.text`包中定义。`java.util.Date`同时包含日期和时间，而`java.sql.Date`仅包含日期，将其纳入`java.sql`包并不合理。另外这两个类都有相同的名字，这本身就是一个非常糟糕的设计。

 - 时区处理麻烦 − 日期类并不提供国际化，没有时区支持，因此`Java`引入了`java.util.Calendar`和`java.util.TimeZone`类，但他们同样存在上述所有的问题。

`Java 8 `在` java.time `包下提供了很多新的 `API`。以下为两个比较重要的` API`：

 - `Local(本地) `− 简化了日期时间的处理，没有时区的问题。

 - `Zoned(时区)` − 通过制定的时区处理日期时间。

新的`java.time`包涵盖了所有处理日期，时间，日期/时间，时区，时刻`（instants）`，过程`（during）`与时钟`（clock）`的操作

### 本地化日期时间 API

`LocalDate/LocalTime `和` LocalDateTime `类可以在处理时区不是必须的情况。代码如下：

```java
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.LocalDateTime;
import java.time.Month;
 
public class Java8Tester {
   public static void main(String args[]){
      Java8Tester java8tester = new Java8Tester();
      java8tester.testLocalDateTime();
   }
    
   public void testLocalDateTime(){
    
      // 获取当前的日期时间
      LocalDateTime currentTime = LocalDateTime.now();
      System.out.println("当前时间: " + currentTime);
        
      LocalDate date1 = currentTime.toLocalDate();
      System.out.println("date1: " + date1);
        
      Month month = currentTime.getMonth();
      int day = currentTime.getDayOfMonth();
      int seconds = currentTime.getSecond();
        
      System.out.println("月: " + month +", 日: " + day +", 秒: " + seconds);
        
      LocalDateTime date2 = currentTime.withDayOfMonth(10).withYear(2012);
      System.out.println("date2: " + date2);
        
      // 12 december 2014
      LocalDate date3 = LocalDate.of(2014, Month.DECEMBER, 12);
      System.out.println("date3: " + date3);
        
      // 22 小时 15 分钟
      LocalTime date4 = LocalTime.of(22, 15);
      System.out.println("date4: " + date4);
        
      // 解析字符串
      LocalTime date5 = LocalTime.parse("20:15:30");
      System.out.println("date5: " + date5);
   }
}
```
执行以上脚本，输出结果为：

```java
当前时间: 2016-04-15T16:55:48.668
date1: 2016-04-15
月: APRIL, 日: 15, 秒: 48
date2: 2012-04-10T16:55:48.668
date3: 2014-12-12
date4: 22:15
date5: 20:15:30
```

### 使用时区的日期时间API

如果我们需要考虑到时区，就可以使用时区的日期时间API：

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;
 
public class Java8Tester {
   public static void main(String args[]){
      Java8Tester java8tester = new Java8Tester();
      java8tester.testZonedDateTime();
   }
    
   public void testZonedDateTime(){
    
      // 获取当前时间日期
      ZonedDateTime date1 = ZonedDateTime.parse("2015-12-03T10:15:30+05:30[Asia/Shanghai]");
      System.out.println("date1: " + date1);
        
      ZoneId id = ZoneId.of("Europe/Paris");
      System.out.println("ZoneId: " + id);
        
      ZoneId currentZone = ZoneId.systemDefault();
      System.out.println("当期时区: " + currentZone);
   }
}
```

执行以上脚本，输出结果为：

```java
date1: 2015-12-03T10:15:30+08:00[Asia/Shanghai]
ZoneId: Europe/Paris
当期时区: Asia/Shanghai
```
（补充）ZoneId.of()里面参数为时区名称，可以通过ZoneId.getAvailableZoneIds()来查看全部时区名称。

Optional 类
----------

`Optional `类是一个可以为`null`的容器对象。如果值存在则`isPresent()`方法会返回`true`，调用`get()`方法会返回该对象。

`Optional `是个容器：它可以保存类型T的值，或者仅仅保存`null。Optional`提供很多有用的方法，这样我们就不用显式进行空值检测。

`Optional `类的引入很好的解决空指针异常。

### 类声明

以下是一个` java.util.Optional<T>` 类的声明：

```java
public final class Optional<T>
extends Object
```

### Optional 实例

我们可以通过以下实例来更好的了解` Optional `类的使用：

```java
import java.util.Optional;
 
public class Java8Tester {
   public static void main(String args[]){
   
      Java8Tester java8Tester = new Java8Tester();
      Integer value1 = null;
      Integer value2 = new Integer(10);
        
      // Optional.ofNullable - 允许传递为 null 参数
      Optional<Integer> a = Optional.ofNullable(value1);
        
      // Optional.of - 如果传递的参数是 null，抛出异常 NullPointerException
      Optional<Integer> b = Optional.of(value2);
      System.out.println(java8Tester.sum(a,b));
   }
    
   public Integer sum(Optional<Integer> a, Optional<Integer> b){
    
      // Optional.isPresent - 判断值是否存在
        
      System.out.println("第一个参数值存在: " + a.isPresent());
      System.out.println("第二个参数值存在: " + b.isPresent());
        
      // Optional.orElse - 如果值存在，返回它，否则返回默认值
      Integer value1 = a.orElse(new Integer(0));
        
      //Optional.get - 获取值，值需要存在
      Integer value2 = b.get();
      return value1 + value2;
   }
}
```

执行以上脚本，输出结果为：

```java
第一个参数值存在: false
第二个参数值存在: true
10
```
`Optional`类更多方法可查看`API`，这里不做记录。


[1]: http://www.runoob.com/java/java8-new-features.html
[2]: https://docs.oracle.com/javase/8/docs/api