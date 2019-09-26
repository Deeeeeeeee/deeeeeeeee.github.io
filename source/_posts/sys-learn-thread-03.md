---
title: 【系统学习】【三】并发编程-原子操作
categories:
  - java 学习
tags:
  - Java
  - thread
date: 2019-09-26 16:52:08
---

## 概要

今晚要去打球了，就简单聊一下原子操作吧，这天气打球不怎么出汗，还是挺舒服的。并发编程很重要的是要解决共享数据带来的冲突问题，而 Java 自带的 synchronized 是阻塞的锁，有时候线程数上来的时候，简单的操作就不适合 synchronized 加锁了。有一个很重要的概念叫 CAS，这个是非阻塞的，在很多地方都有利用这个概念提高效率。不过一般情况下，都推荐使用 synchronized，因为 jvm 做了很多的优化，而且不容易出错。所以今天就聊聊以下几个方面

- synchronized 可能带来的问题
- CAS 原理
- CAS 可能带来的问题
- JDK 中的 CAS 原子操作

<!-- more -->

## synchronized 可能带来的问题

因为是阻塞的并且是互斥的，则需要考虑以下几个问题

- 出现饥饿
  - 永远阻塞在同步代码块中，因为有一个线程要花很长时间去执行同步代码块
  - 高优先级的线程完全占用了低优先级线程的CPU时间片
  - wait 其他线程但永远不被唤醒，因为一直在唤醒其他线程
- 拿到锁的线程一直不释放锁，其他线程一直被阻塞
- 大量竞争的时候会消耗很多CPU资源，会带来死锁或其他安全性问题

还有一个活锁的概念：两个线程没有阻塞，但一直在让对方先执行，进入了死循环。可以理解为互相谦让，然后一直互相谦让

[这里有一篇介绍死锁、活锁和饥饿的文章](https://www.codejava.net/java-core/concurrency/understanding-deadlock-livelock-and-starvation-with-code-examples-in-java)

## CAS 原理

现代计算机有一个指令 cmpxchg(Compare and Exchange)，这个是原子性操作，Java 的 CAS(Compare and Swap 或者 Compare and Set) 利用的就是这个特性

CAS 的概念其实也很简单，但是很重要，是构成 jdk 并发包的基石。**一个内存地址的值V ，期望值A，新值B。当V==A时，赋值为B，否则不做操作**

## CAS 可能带来的问题

- ABA 问题
  - 产生的原因：如果有两个线程给一个账户扣减5万元，但实际希望只操作一次，正常来说 CAS 可以保证只操作一次。但是在第一个线程执行完时，有第三个线程加了 5 万元，这时候第二个线程再执行，又扣了5万元，这就多扣了
  - 解决：使用版本号解决这个问题。AtomicMarkableReference、AtomicStampedReference 就是带版本戳的
- 开销问题
  - 产生的原因：一般情况下，CAS 操作都会配合自旋使用，自旋就是无限循环(for(;;))，这样会带来 CPU 的消耗
  - 解决：限制自旋的次数
- 只能保证一个共享变量原子操作
  - 解决：多个变量合并成一个变量来解决这个问题，或者使用 synchronized 或显示锁

## JDK 中的 CAS 原子操作

JDK 提供了很多原子操作的类，JDK 中 CAS 可以分为以下几类

- 更新基本类型类：AtomicBoolean、AtomicInteger、AtomicLong
- 更新数组类：AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
- 更新引用类型类：AtomicReference、AtomicMarkableReference、AtomicStampedReference
- 更新字段类：AtomicReferenceFieldUpdater、AtomicIntegerFieldUpdater、AtomicLongFieldUpdater

常用的 AtomicBoolean、AtomicInteger、AtomicLong、AtomicReference、AtomicMarkableReference、AtomicStampedReference

这里 AtomicMarkableReference、AtomicStampedReference 带版本戳，解决 ABA 问题的类。他们的区别是

- AtomicMarkableReference，boolean 关心是否动过
- AtomicStampedReference，integer 动过几次。通过比较 stamp 实现

```java
// jdk 里面 Excecutors 通过 AtomichInteger 记录 poolNumber 和 threadNumber
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                      "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

这一篇说的比较简单，其实 java 的 CAS 实现都是 sun.misc.Unsafe 里面的本地方法实现的。主要还是要理解这个 CAS 概念，其实概念也很容易理解。。。平时开发的时候，如果用 synchronized 或显示锁遇到有效率问题时，可以考虑使用 CAS，毕竟非阻塞