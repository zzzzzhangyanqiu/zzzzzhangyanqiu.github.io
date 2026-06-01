---
title: Java线程中断机制（interrupt）
tags:
  - Java
date: 2020-12-19
categories:
 - Java
---

#### 前言

现在我们在使用`Thread`对象执行代码的时候，如果你比较细心，一定会发现，像是`Thread.stop()、Thread.suspend()、Thread.resume()`这几个方法已经废弃了，而官方的中断线程推荐做法是让你调用`Thread.interrupt()`方法，那么为什么会有这种情况呢，下面来分析一下。

#### 暴力的中断线程

其实，官方废弃掉上述三个方法的原因，主要是因为这三个方法太“暴力”了。也就是说，当你调用这几个方法的时候，不管当前线程有没有执行完，适不适合中断，如果中断后会不会出现问题，它都会强制执行。这样就会使程序的运行结果和我们想的不一样，来看下面的一个例子。

```java
    public static void main(String[] args) throws InterruptedException {
        MyThread thread = new MyThread();
        thread.start();
        
        //先执行一秒钟
        thread.sleep(1000);
        
        thread.stop();


        // 确保线程已经销毁
        while (thread.isAlive()) { }

        // 输出结果
        thread.print();
    }


    private static class MyThread extends Thread {
        private int x = 0;

        private int y = 0;

        @Override
        public void run() {
            synchronized (this) {
                x++;
                //模拟业务操作
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                y++;
            }
        }


        public void print() {
            System.out.println("x:" + x + "\ty:" + y);
        }
    }
```

以上代码的输出结果为`x:1	y:0`

你会发现，这里面与我们预想的运行结果是不一样的，因为它是直接暴力的将线程给干掉了，不管代码是否执行完。可以预料到，这种策略在我们实际业务中肯定会出现问题，所以后来官方新增了线程停止方法`Thread.interrupt()`

#### 优雅的中断线程

首先看如下代码

```java
    public static void main(String[] args) throws InterruptedException {
        MyThread thread = new MyThread();
        thread.start();

        //先执行一秒钟
        thread.sleep(1000);

        //注意，这里将stop更换成了interrupt
        thread.interrupt();

        // 确保线程已经销毁
        while (thread.isAlive()) { }

        // 输出结果
        thread.print();
    }


    private static class MyThread extends Thread {
        private int x = 0;

        private int y = 0;

        @Override
        public void run() {
            synchronized (this) {
                x++;
                //模拟业务操作
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                y++;
            }
        }

        public void print() {
            System.out.println("x:" + x + "\ty:" + y);
        }
    }
```

以上代码的运行结果如下

```java
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.rongda.concurrent.interrupt.InterruptTest$MyThread.run(InterruptTest.java:48)
x:1	y:1
```

可以看到，结果是符合我们预期的，并且还抛出了异常，`interrupt()`的设计思想是：线程应该由自身来决定是否停止，其它线程没有权限来操作。所以根据它的设计思想再加上源码，我们可以了解到<font color=red>**interrupt()并不会中断线程，只是设置一个标志位，线程定时的去检查这个标志位，如果为中断状态，则中断当前线程**</font>。

需要注意的是，在线程处于`WAITING、TIMED_WAITING`状态时，调用`interrupt()`方法会直接抛出异常，抛出异常后，线程的中断标志位会由`true`重置为`false`，猜想可能是因为线程处理完异常之后，重新处于了就绪状态，如下代码

```java
    public static void main(String[] args) {
        MyThread thread = new MyThread();
        thread.start();

        thread.interrupt();
    }


    private static class MyThread extends Thread {

        @Override
        public void run() {
            synchronized (this) {
                //模拟业务操作
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    //这里会输出false
                    System.out.println("thread:" + isInterrupted());
                }
            }
        }
    }
```

最后来统一了解`interrupt`相关的几个方法

```java
    /**
    * 中断当前线程
    **/
	public void interrupt() {
        //如果当前不是本线程
        if (this != Thread.currentThread())
            //确定当前正在运行的线程是否具有修改此线程的权限。
            checkAccess();

        synchronized (blockerLock) {
            
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // 这是一个native方法  仅仅是设置一个中断标志位
                //这行代码的大概意思是，如果有其它操作注册到了此线程上（如I/O操作）
                //在中断此线程的时候，同时要将注册的操作中断（如I/O操作）
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }

    /**
     * 测试当前线程是否已被中断，注意！该方法可以清除线程的中断状态
     * 也就是说，如果此方法被连续调用两次，则第二次调用将返回false（除非当前线程在第一次调用清除其中断的状态之后再次被中断，并且      * 在第二次调用检查之前）
     */
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }

    /**
     * 测试此线程是否已被中断。线程的中断状态不受此方法的影响
     */
    public boolean isInterrupted() {
        return isInterrupted(false);
    }

    /**
     * 测试当前线程是否已被中断。中断状态是否根据传递的ClearInterrupted值决定是否重置
     */
    private native boolean isInterrupted(boolean ClearInterrupted);
```



