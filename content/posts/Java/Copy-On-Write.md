---
title: Copy On Write
tags:
  - Java
date: 2018-06-05
categories:
 - Java
---
> 写入时复制，它包含下面两个命令
## fork: ##

百度百科翻译为:

`UNIX`及类`UNIX（UNIX-like）`系统中的分叉函数。返回值： 若成功调用一次则返回两个值，子进程返回0，父进程返回子进程标记；否则，出错返回-1。
`fork`函数将运行着的程序分成2个（几乎）完全一样的进程，每个进程都启动一个从代码的同一位置开始执行的线程。这两个进程中的线程继续执行，就像是两个用户同时启动了该应用程序的两个副本。
`Linux`中,所有进程都是`init`进程的"儿子"，即所有进程均是由`init`进程衍生出来，类似于`Java`中的`object`类。具体做法为：当有程序需要进行创建进程执行时，首先复制`init`进程(除去进程`ID`的全部状态，包括指针，内存空间等)，之后再把数据全部抹去，填充具体程序的内存数据。此为`fork()`。

## exec: ##

在`Linux`中，并不存在一个`exec()`的函数形式，`exec`指的是一组函数，一共有6个，分别是：

```c
#include <unistd.h>
extern char **environ;
int execl(const char *path, const char *arg, ...);
int execlp(const char *file, const char *arg, ...);
int execle(const char *path, const char *arg, ..., char *const envp[]);
int execvp(const char *file, char *const argv[]);
int execve(const char *path, char *const argv[], char *const envp[]);
int execv(const char *path, char *const argv[]);
```

其中只有`execve`是真正意义上的系统调用，其它都是在此基础上经过包装的库函数。

由于fork函数需要将所有的数据都复制到新的进程中，之后又将其抹去，其中的过程比较耗费资源，所有Linux对其做了优化------每当有程序需要`fork`进程时，`Linux`会将进程创建一个备份A，A为只读区域，然后将此进程指向备份的内存区域，此进程只保留指针，最后再复制此进程，这时，每个进程保留的只是指针，不需要进行全部复制，所以速度会快很多。
如果有程序要再此进程进行新的任务，首先进程会修改它所指向的内存区域A，发现A为只读，系统会抛出异常，在异常处理中将该进程中，程序申请的内存页清空，随后设置新的内存数据，此过程为`exec()`;

## 在JAVA中的应用: ##

### CopyOnWriteArrayList： ###

`JAVA`中，`ArrayList`为线程不安全的，如果想使用线程安全的`List`类，`JAVA`已经为我们提供好了`Vector`，观察`Vector`的代码，发现几乎所有的方法都是使用`synchronized`来实现的，使用`synchronized`就意味着在数据量大的时候必然会影响效率，而且只是单独方法进行了`synchronized`。如果有两个线程，A线程在`clear()`的同时，B线程正在`get()`，那必然会发生错误，那有没有一个解决方案呢。
在`java.util.concurrent`包中，`JAVA`已经为我们提供好了一个类`CopyOnWriteArrayList`，观看此类的源码，会发现该类的原理就是在set的时候先复制一个同样的数组，然后修改新的数组，最后在`set`到原数组中。`get`的时候不是返回原数组而是返回一个原数组的副本。

#### 注意事项 ####

需要注意的是，由于此类特殊的set方式，因此效率相对较低，测试一下`10W`条数据的场景：

```java
/**
 * @author zhangyq
 */
public class Test {

    public static void main(String[] args) {
        List array = new ArrayList();

        long arrayTime = System.nanoTime();

        for(int i = 0;i < 10 * 10000;i++){
            array.add(i);
        }

        long arrayTime1 = System.nanoTime();

        List list = new CopyOnWriteArrayList();

        long copyArrayTime = System.nanoTime();

        for(int i = 0;i < 10 * 10000;i++){
            list.add(i);
        }

        long copyArrayTime1 = System.nanoTime();

        System.out.println("ArrayList消耗时间为:" + (arrayTime1 - arrayTime));

        System.out.println("CopyOnWriteArrayList消耗时间为:" + (copyArrayTime1 - copyArrayTime));

    }
}
```

运行结果为：
	
```java
ArrayList消耗时间为:5511000
CopyOnWriteArrayList消耗时间为:5134985100
```

当然，`CopyOnWriteArrayList`的优势是多线程，这样测试难免会有些"不公平"，结果仅仅作为参考即可。使用此类时要注意性能问题，尽量保持写少读多。