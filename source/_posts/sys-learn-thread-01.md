---
title: 【系统学习】【一】并发编程
categories:
  - java 学习
tags:
  - Java
  - thread
date: 2019-09-19 16:18:18
---

有很长一段时间没有写博客，刚好最近对Java体系有系统学习的想法，有机会也可以总结一下这段时间以来的成长，虽然一直坚持写可能有更好的收获

## 概要

并发编程分几个部分
- 基础概念
- Java 里的线程

<!-- more -->

## 基础概念

### CPU 核心数和线程数

一直以来 CPU 核心数和线程数是 1 比 1 的关系，像 2006 年 Intel 发布的 Core2 双核处理器。2002 年 Intel 发布奔腾4处理器，推出其创新超线程(hyperthreading) 技术，使得核心数与线程数变成 1 比 2 的关系。像如今的 i9-9900 就是八核十六线程的

### CPU 时间片轮转机制

时间片轮转调度是一种最古老，最简单，最公平且使用最广的算法，又称RR调度。每个进程被分配一个时间段，称作它的时间片，即该进程允许运行的时间。基本上可以理解为给队首分配一定的时间片，然后丢到队尾。

进程间的切换是需要一定时间的，保存和装入寄存器值及内存映像，更新各种表格和队列等，称为上下文切换

### 进程和线程

进程是操作系统对一个正在运行的程序的一种抽象，也是程序运行进行资源分配的最小单位。进程之间资源是相互独立的

线程运行在进程的上下文中，同一进程下的多个线程共享同样的代码和全局数据，也是 CPU 调度的最小单位

### 并行和并发

- 并发指一定时间内处理多个事情的能力
- 并行指同一时间可以处理多个事情的能力

一边打电话一边吃饭就是并行

打完电话，吃饭，洗碗，这段时间内做的事情就是并发能力

### 并发编程需要解决的问题

- 线程共享资源，存在冲突
- 死锁
- 线程数太多，导致资源消耗殆尽

## JAVA 里的线程

### 线程管理接口

ThreadMXBean。Java 天生就是多线程，启动多个守护线程

```java
public static void main(String[] args) {
    ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
    ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
    for (ThreadInfo info : threadInfos) {
        System.out.println(info.getThreadName());
    }
}

// 上面输出的结果
Monitor Ctrl-Break
Signal Dispatcher
Finalizer
Reference Handler
main
```

### 启动线程的方式

- 继承 Thread 类
- 实现 Runnable 接口
- 实现 Callable 接口

但 new Thread() 不接收 Callbale 接口，所以需要 FutureTask 进行包装

```java
public static class CallableClazz implements Callable<String> {
    @Override
    public String call() throws Exception {
        System.out.println("Called.");
        return "CallResult.";
    }
}

public static void main(String[] args) throws ExecutionException, InterruptedException {
    CallableClazz clazz = new CallableClazz();
    FutureTask<String> futureTask = new FutureTask<>(clazz);
    Thread thread = new Thread(futureTask);
    thread.start();
    System.out.println(futureTask.get());
}
```

稍微看一下 FutureTask

- FutureTask 实现 Runnable 接口

{% asset_img futureTask.png %}

- FutureTask 的 run 方法

```java
// 可以看到在 run 方法里面执行了 call 方法
public void run() {
    ...
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        ...
    }
}
```

### 线程停止

- 运行完毕
- 抛出异常
- 主动停止
  - 废弃方法：stop()、resume()、suspend()。因为不能保证资源的释放，容易造成程序不可知问题和死锁问题
  - 新方法
    - interrupt() 中断线程
    - isInterrupted() 判定当前线程是否中断状态
    - interrupted() static 方法，判定当前线程是否中断状态。会清除中断状态，即改为 false，这是最大的区别

```java
// 如果连续调用这个方法，第二次会返回 false
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}

public boolean isInterrupted() {
    return isInterrupted(false);
}

/**
    * Tests if some Thread has been interrupted.  The interrupted state
    * is reset or not based on the value of ClearInterrupted that is
    * passed.
    */
private native boolean isInterrupted(boolean ClearInterrupted);
```

#### Java 线程是协作式线程，调用了 interrupt() 不一定会中断线程

```java
// 这里调用了 interrupt 方法，但没有根据中断状态进行处理，是不会停止的
public static class TreadClazz extends Thread {
    @Override
    public void run() {
        while (true) {
            System.out.println("Running...");
        }
    }
}

public static void main(String[] args) {
    TreadClazz clazz = new TreadClazz();
    clazz.start();
    clazz.interrupt();
}
```

#### 当线程方法抛出 InterruptedException时，线程中断标志位会复位成 false

```java
    /**
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public static native void sleep(long millis) throws InterruptedException;
```

这时候如果在线程运行时，抛出 InterruptedException，而没有重新将中断标志为改为 true，是不会停止的

```java
// 只要在 catch 里再调用一下 interrupt() 方法，中断标志位就会改为 true，从而退出循环
public static class TreadClazz extends Thread {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                System.out.println("threadName:" + Thread.currentThread().getName() + ". state is " +
                        Thread.currentThread().isInterrupted());
                e.printStackTrace();
            }
            System.out.println("Running...");
        }
        System.out.println("threadName:" + Thread.currentThread().getName() + " interrupt");
    }
}

public static void main(String[] args) throws InterruptedException {
    TreadClazz clazz = new TreadClazz();
    clazz.start();
    Thread.sleep(100);
    clazz.interrupt();
}
```

### Java 线程

#### Java 线程状态

- NEW
-  RUNNABLE
- BLOCKED
- WAITING
- TIMED-WAITING
- TEMINATED

{% asset_img thread-lifecycle.jpeg %}

#### run 和 start 方法

直接调用 run 方法跟调用一个对象的普通方法没有任何区别，不会让虚拟机创建线程

#### 线程的优先级

- 优先级分 1~10 个等级，默认等级为 5，越小越高，cpu分配时间片会越多
- 但是操作系统可能没有理会这个优先级，所以不一定会有效

#### 守护线程

- 主线程死亡，守护线程也死亡。例如 GC 线程
- 注意 try finally 对守护线程无效

### Java 线程间共享

#### synchronized 内置锁

Java 内置锁分为对象锁和类锁

- 多个线程获取的锁是不同的对象，是可以同时运行的；多个线程获取的锁是同一个对象，是一个接着一个运行的
- 类锁锁的是 Class 对象
- synchronized 修饰 static 方法是类锁

```java
// 类锁
public synchronized static void increase() {
    aInt++;
}

public void increase2() {
    synchronized (SynchronizedEG.class) {
        bInt++;
    }
}

// 对象锁
public synchronized void increase3() {
    bInt++;
}

public void increase4() {
    synchronized (this) {
        bInt++;
    }
}
```

#### volatile 关键字

- 读取时从主内存读取；设置时刷新到主内存
- 没有使用锁，是最轻量级的同步机制，非线程安全
- 使用场景，只有一个线程写，多个线程读

```java
// volatile 不保证原子性
public static class TreadClazz extends Thread {
    @Override
    public void run() {
        aInt++;
        System.out.println("threadName:" + Thread.currentThread().getName() + ". aInt is " + aInt);
        try {
            Thread.sleep(40);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        aInt++;
        System.out.println("threadName:" + Thread.currentThread().getName() + ". aInt is " + aInt);
    }
}

public static void main(String[] args) {
    TreadClazz clazz0 = new TreadClazz();
    TreadClazz clazz1 = new TreadClazz();
    TreadClazz clazz2 = new TreadClazz();
    TreadClazz clazz3 = new TreadClazz();

    clazz0.start();
    clazz1.start();
    clazz2.start();
    clazz3.start();
}

// 输出结果
threadName:Thread-0. aInt is 1
threadName:Thread-1. aInt is 1
threadName:Thread-3. aInt is 3
threadName:Thread-2. aInt is 3
threadName:Thread-0. aInt is 4
threadName:Thread-1. aInt is 5
threadName:Thread-3. aInt is 6
threadName:Thread-2. aInt is 7
```

#### ThreadLocal

- 用空间换线程的安全性
- 线程级别变量隔离
- 主键为线程，值为需要存的值