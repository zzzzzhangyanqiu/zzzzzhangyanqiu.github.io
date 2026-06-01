---
title: 线程本地变量-ThreadLocal
tags:
  - Java
date: 2021-01-02
categories:
 - Java
---

### 前言
`JDK 1.2`的版本中就提供`java.lang.ThreadLocal`，`ThreadLocal`为解决多线程程序的并发问题提供了一种新的思路。使用这个工具类可以很简洁地编写出优美的多线程程序，<font color=red>`ThreadLocal`并不是一个`Thread`，而是`Thread`的局部变量</font>。需要注意的是，因为`ThreadLocal`的命名，很多人会认为这是个“本地线程”，但是它实际上是个变量，用来存储当前线程在此变量中的值。

由于`ThreadLocal`使用了`Java`中的弱引用，所以在了解`ThreadLocal`之前，先让我们了解一下`Java`中的四种引用类型

#### 强引用

顾名思义，就是最强的引用，我们平时使用的最多的引用，只要引用关系还存在，那么就永远都不会被`GC`（即使抛出了`OOM`）

如下代码

```java
    public static void main(String[] args) throws IOException {
        Users users = new Users();
        //如果引用存在，那么永远都不可能被回收
        //这里将引用置空
        users = null;
        System.gc();
        //需要注意这句，由于gc是由一个单独的线程（gc线程）来做的
        //所以可能main线程等不到gc就结束了，所以这里要一直阻塞住
        System.in.read();
    }

    private static class Users {

        public String name;

        public Users() {
        }

        @Override
        protected void finalize() {
            //在这里标记对象已经被回收了
            System.out.println("users finalize");
        }
    }
```

#### 软引用

软引用用来描述一些**有用但并不是必需**的对象，在`Java`中由`SoftReference`表示，软引用只有在`jvm`内存不足时才会被回收，查看如下代码

```java
    public static void main(String[] args) throws IOException {
        //首先将jvm堆大小设置为15m，便于垃圾回收，-Xmx15m -Xms15m
        //软引用由SoftReference表示，通过soft.get()获取存入的值
        //这里创建一个10M内存大小的数组
        SoftReference<byte[]> soft = new SoftReference<>(new byte[1024 * 1024 * 10]);
        //首先第一次并不会被回收
        System.out.println(soft.get());
        System.gc();
        //第二次也不会被回收，因为前面说过了，只有在内存不足时才会被回收
        System.out.println(soft.get());
        //创建一个新的10M大小的数组，此时jvm堆空间已经不够了，所以会回收前面的软引用
        byte[] bytes = new byte[1024 * 1024 * 10];
        //此时已经被回收了，所以这里是null
        System.out.println(soft.get());
        //软引用本身
        System.out.println(soft);
        System.in.read();
    }
```

以上代码的运行结果为

```java
[B@7cca494b
[B@7cca494b
null
java.lang.ref.SoftReference@7ba4f24f
```

可以看到，虽然软应用内的对象已经被回收了，但是指向软引用本身的指针还没有被回收，因为<font color=red>软引用本身是强引用</font>，<font color=red>软引用本身和软引用指向的对象不是同一个！！</font>这样说可能会有些绕，看下图

![image-20201210173551854](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/picgo/image-20201210173551854.png)

##### 应用场景

由于软引用只有在`jvm`内存不足时才会被回收，因此，这一点可以很好地用来解决`OOM`的问题，所以就特别适合用来做缓存

#### 弱引用

弱引用用来描述一些**永远不是必需**的对象，在`Java`中由`WeakReference`表示，只要`jvm`发生`GC`，弱引用就会被回收，查看如下代码

```java
    public static void main(String[] args) throws IOException {
        //创建弱应用
        WeakReference<byte[]> bytes = new WeakReference<>(new byte[1024 * 1024 * 10]);
        //第一次输出
        System.out.println(bytes.get());
        System.gc();
        //GC后第二次输出
        System.out.println(bytes.get());
        System.in.read();
    }
```

以上代码输出结果为

```java
[B@7cca494b
null
```

需要注意的是，弱引用和软引用一样（包括后面的虚引用），即使引用的对象被回收了，但是对象本身还在

弱引用还可以配合引用队列`WeakReferenceQueue`使用，这点和后面的虚引用一样，所以放在一起说

##### 应用场景

弱引用通常用来解决一些内存泄露的问题，如后面讲到的`ThreadLocal`使用的就是弱引用

#### 虚引用

虚引用在`java`中用`PhantomReference`表示，如果一个对象和虚引用关联，那就相当于没有引用，随时都有可能被`jvm`回收。虚引用永远都获取不到引用的对象。虚引用必须要和引用队列关联使用，在引用的对象被回收之后，`jvm`会把回收的对象加入引用队列中，我们可以监听引用队列来进行其它的操作，查看如下代码

```java

    private static List<Object> TEST_DATA = new LinkedList<>();
    /**
     * 创建引用队列
     * */
    private static ReferenceQueue<Users> queue = new ReferenceQueue<>();

    public static void main(String[] args) throws IOException {
        //首先将jvm堆大小设置为15m，便于垃圾回收，-Xmx15m -Xms15m
        //创建虚引用
        PhantomReference<Users> o = new PhantomReference<>(new Users("zhangsan"), queue);

        //获取一次，虚引用无法获取到对象实例，这里是null
        System.out.println(o.get());	//null

        //占满内存，促使jvm尽快GC
        new Thread(() -> {
            while (true) {
                TEST_DATA.add(new byte[1024 * 1024]);
            }
        }).start();

        //监听引用队列
        new Thread(() -> {
            while (true) {
                Reference<? extends Users> poll = queue.poll();
                if (poll != null) {
                    //这里是null，由于虚引用无法获取到对象实例，这里只用于监听
                    System.out.println(poll + ":已经被回收");	//java.lang.ref.PhantomReference@62661526:已经被回收
                }
            }
        }).start();

        System.in.read();
    }

    private static class Users {
        public String name;

        public Users(String name) {
            this.name = name;
        }
    }
```

以上代码输出结果为

```java
null
java.lang.ref.PhantomReference@62661526:已经被回收
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at com.reference.PhantomReferenceTest.lambda$main$0(PhantomReferenceTest.java:44)
	at com.reference.PhantomReferenceTest$$Lambda$1/610998173.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
```

##### 应用场景

看到这里大家可能会有一些疑问，虚引用跟没有一样，那我们要它干什么呢。其实虚引用最大的作用就是管理堆外内存，由于`jvm`不能回收堆外内存，所以代码中可以保存一个堆外内存的引用对象，当这个引用对象被回收的时候，通过监听引用队列，我们就可以手动的把堆外内存释放掉。那么为什么有了`JVM`内存，我们还需要堆外内存呢？简单来说，计算机处理网络请求时，首先要先将数据从网卡读取到计算机内存，然后再从计算机内存复制到`JVM`内存，（里面还涉及到用户态和内核态的切换、缓冲区等，这里为了简单不详细描述）这个复制过程非常浪费资源，如果可以直接使用计算机内存，那么就不需要复制了，性能也会提升（这也是`netty`比较高性能的原因之一），如下图

![image-20201210193156284](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/picgo/image-20201210193156284.png)

到这里，铺垫工作就完成了，下面我们就正式开始介绍今天的大菜——`ThreadLocal`

### ThreadLocal

#### 使用

```java

    private static ThreadLocal <String> threadLocal = new ThreadLocal<>();
    private static ThreadLocal <String> threadLocal2 = new ThreadLocal<>();

    /**
     * 此处可以重写initialValue为当前线程提供默认值
     * **/
//    private static ThreadLocal <String> threadLocal = new ThreadLocal<String>(){
//        @Override
//        protected String initialValue() {
//            return "默认值";
//        }
//    };

    public static void main(String[] args) {
        new Thread(() -> {
            threadLocal.set(Thread.currentThread().getName() + "--threadA");
        }).start();

        new Thread(() -> {
            try {
                //避免threadA还没有执行set方法，延迟两秒钟
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("threadB：" + threadLocal.get());	//threadB：null
        }).start();

   }
```

以上代码运行结果为：

```java
threadB：null
```

可以看到，即使在`threadA`中对此变量赋值了，也不会影响到线程`B`的取值，两个线程之间变量是隔离的。

#### 实际应用

使用`SimpleDateFormat`时，因为它里面的某些变量是`static`的，在多线程环境下及其容易发生数据冲突，这种情况就可以使用`ThreadLocal`为每个线程都单独创建一个`SimpleDateFormat`对象

`Spring`中的事务管理也是使用的`ThreadLocal`

#### 源码分析

古人云：取其精华去其糟粕，所以我们这里只需要看关键的几处代码就可以了，对于我们的示例代码来说，主要的方法就两个

1. `threadLocal.set()`
2. `threadLocal.get()`

首先我们来看`set()`方法

```java
  public void set(T value) {
        //首先获取到当前线程
        Thread t = Thread.currentThread();
        //根据线程获取到ThreadLocalMap对象
        //暂时先当做一个普通的map集合  同样存储K，V键值对
        ThreadLocalMap map = getMap(t);
        //赋值
        if (map != null)
            //注意  这里的this指的是threadLocal对象
            map.set(this, value);
        else
            createMap(t, value);
    }

    ThreadLocalMap getMap(Thread t) {
        //注意   这里取的是Thread类里面的一个成员变量 threadLocals
        return t.threadLocals;
    }
```

上述代码大概的流程为

1. 获取线程对象里面的一个成员变量`threadLocals`，该成员变量是一个`map`集合
2. 操作上一步获取到的`map`集合，如果为空则新建，如果不为空则赋值，赋值的`key`为`threadLocal`对象

看到这里大家应该明白了。害，怪不得`threadLocal`可以做到线程隔离，因为人家就是存到了`thread`对象里面

这样虽然方便，但是有一个问题，不知道大家想过没有，就是我们最开始`new`出来的`threadLocal`对象，好像永远都不可能被回收了，因为就算外部的引用已经被置为了`null`，那只要有一个线程是存活的，就是一次强引用，就不可能被回收，如图所示

![image-20201210195357229](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/picgo/image-20201210195357229.png)

龟龟，这要是时间长了不是要发生内存泄露了么，别慌，带着这个问题，我们一起来看`ThreadLocal`中的核心数据结构`ThreadLocalMap`

```java
public class ThreadLocal<T> {
    
    //省略其它代码
    static class ThreadLocalMap {
		/**
		* Entry就是ThreadLocalMap中的每一个元素（跟hashmap一样）
		* 注意看  这里继承了WeakReference，并且把key传到了弱引用里面
		*/
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** ThreadLocal 存储的值 */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        /**
         * 赋值
         */
        private void set(ThreadLocal<?> key, Object value) {
           	//省略其它代码
            tab[i] = new Entry(key, value);
            //省略其它代码
        }

    }
}  
```

如果`ThreadLocalMap`中存储的是弱引用，那么关系图就会变成这样

![image-20201210200201506](https://suiyueranzly.oss-cn-beijing.aliyuncs.com/picgo/image-20201210200201506.png)

`OK`，现在`key`已经不存在引用关系了，但是还是有一个问题。`ThreadLocalMap`中存储的`key`都是`null`了，那我的`value`不是永远访问不到了？龟龟，时间一长，还是要发生内存泄露的啊。别急，这个问题，我们等到最后一起解答。

其实到了这里，`get()`方法就不需要再分析了，简单来说，就是获取到`Thread`对象里面的`threadLocals`字段，在从`map`集合中直接`get()`就可以了

```java
    public T get() {
        //获取当前线程
        Thread t = Thread.currentThread();
        //获取线程内变量
        ThreadLocalMap map = getMap(t);
        //如果不等于null则直接get()返回
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                T result = (T)e.value;
                return result;
            }
        }
        //如果等于null，就设置并返回默认值
        return setInitialValue();
    }
```

到这里，`ThreadLocal`的主要思想就说完了，但是还是有很多细节，比如`ThreadLocalMap`中`get()/set()/remove()`方法的实现，这部分就不深究了，感兴趣的同学可以自行学习

#### 注意

前面埋下了一个伏笔，就是关于`ThreadLocalMap`中`key`为`null`时，`value`的回收情况，除了这个问题，在使用`ThreadLocal`的时候，还需要注意以下两点

脏数据：如果你使用的线程池的方式，因为线程池中的线程是可以复用的，那么可能`ThreadLocal.get()`方法有的时候会超出你的预计，因为它可能会获取到上一个线程中赋的值，造成数据错误。

内存泄漏：`ThreadLocal`注释中推荐将此对象声明为`static`方式，但是这样的话，寄希望于弱引用去清除值就显得不现实了（因为`ThreadLocal`的引用永远都不会被中断），于是数据就会一直堆积，这种情况下容易发生内存泄漏。

以上三种情况，均可以在使用完毕后，调用`remove()`方法，移除变量中的数据。