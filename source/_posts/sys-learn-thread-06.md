---
title: 【系统学习】【六】并发编程-线程池
categories:
  - java 学习
tags:
  - Java
  - thread
date: 2019-10-10 17:35:48
---

## 概要

线程池其实之前已经发过一篇了，之前那篇聊的比较细节，这次就泛泛地聊一下。目前我多线程开发，基本都不直接创建 Thread 进行操作，而是使用线程池去完成任务，因为线程池有几大好处。所以几天就聊一下以下几个方面

- 线程池的优势
- 线程池的实现原理
- ThreadPoolExecutor
- jdk 中预定义的线程池
- 合理配置线程池

<!-- more -->

## 线程池的优势

由于线程的创建和销毁是比较消耗资源的，如果可以重复利用创建的线程，那么就可以提高程序的性能。那么我认为线程池的优势有以下几点

- 降低线程创建销毁的资源消耗，因为重复利用已创建好的线程
- 提高响应速度，节约了线程创建和销毁的时间。因为创建和销毁线程也是需要不少时间的，如果直接 new Thread() 的话，这两部分时间就节省不了
- 提高线程的可管理性

## 线程池的实现原理

先聊一下线程池有哪些功能，这里主要说 ThreadPoolExecutor，因为大多数都是使用这个线程池

- 线程创建好，由容器保持
- 线程接收外部任务，并完成任务
- 如果任务较多，由队列容器保持
- 如果任务太多，有饱和策略进行处理
- 任务较少，减少线程
- 取消任务，关闭线程池等管理能力

{% asset_img thread-pool.png %}

这里解释一下 ThreadPoolExecutor 提交任务后的流程

1 corePool 没满，则创建线程去完成任务
2 corePool 满了，则将任务放到阻塞队列中，等待线程获取(take/poll)任务然后执行
3 corePool 满了，阻塞队列也满了，但是 maximumPool 没满，创建线程去完成任务
4 maximumPool 也满了，交给饱和处理器进行处理

## ThreadPoolExecutor

我们来看点源码，ThreadPoolExecutor 的构造方法，提交任务，取消任务，停止线程池，饱和策略

### 构造方法

构造方法有很多个，我们直接看最全的

```java
// 看起来就是一系列的赋值
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

我们看看这些参数有什么作用

- corePoolSize 核心线程数。小于 corePoolSize，创建线程；等于 corePoolSize，任务保存到阻塞队列中
- maximumPoolSize 允许最大线程数。= corePoolSize && <= maximumPoolSize && 阻塞队列也满了，创建线程
- keepAliveTime 和 TimeUnit 线程空闲下来后，存活的时间；默认情况下，这个参数只在 > corePoolSize 的时候才有用
- workQueue 保存任务的阻塞队列
- threadFactory 创建线程的工厂。最主要给线程起名字
- RejectedExecutionHandler
  - AbortPolicy 直接抛出异常，默认的策略
  - CallerRunsPolicy 用调用者所在的线程执行任务
  - DiscardOldestPolicy 丢弃阻塞队列最老的任务
  - DiscardPolicy 当前任务直接丢弃

### 提交任务

提交任务分几种情况处理

- 线程数小于 corePoolSize，添加线程
- 如果大于等于 corePoolSize 则将任务放进队列
- 如果队列满了，放进 maximunPoolSize
- 放失败了使用饱和策略处理

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 线程数小于 corePoolSize，添加线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 如果大于等于 corePoolSize 则将任务放进队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果队列满了，放进 maximunPoolSize
    else if (!addWorker(command, false))
        // 放失败了使用饱和策略处理
        reject(command);
}
```

### 取消任务

我们先来看 ThreadPoolExecutor 的几个变量

```java
// 保存任务的阻塞队列
private final BlockingQueue<Runnable> workQueue;
// 显示锁，当添加线程，中断线程，统计线程数等的时候使用这个锁
private final ReentrantLock mainLock = new ReentrantLock();
// 保持线程的容器，数据结构为 HashSet
private final HashSet<Worker> workers = new HashSet<Worker>();
// 给 awaitTermination 功能使用
private final Condition termination = mainLock.newCondition();
```

线面再来看看取消任务的方法，可以看到正在运行的任务这里不能取消

```java
// 从阻塞队列中移除任务
public boolean remove(Runnable task) {
    boolean removed = workQueue.remove(task);
    tryTerminate(); // In case SHUTDOWN and now empty
    return removed;
}

// 移除所有已经取消的任务
public void purge() {
    final BlockingQueue<Runnable> q = workQueue;
    try {
        Iterator<Runnable> it = q.iterator();
        while (it.hasNext()) {
            Runnable r = it.next();
            if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                it.remove();
        }
    } catch (ConcurrentModificationException fallThrough) {
        // Take slow path if we encounter interference during traversal.
        // Make copy for traversal and call remove for cancelled entries.
        // The slow path is more likely to be O(N*N).
        for (Object r : q.toArray())
            if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                q.remove(r);
    }

    tryTerminate(); // In case SHUTDOWN and now empty
}
```

### 停止线程池

这里有两个方法

- shutdown 设置线程状态外，只会停止所有没有执行任务的线程
- shutdownNow 除了设置线程状态外，还会尝试停止正在运行或者暂时的任务

其实主要的区别是 shutdownNow 设置线程池状态为 STOP，shutdown 设置线程池状态为 SHUTDOWN。shutdown 是可以获取到队列里面任务的，所以线程会继续执行；而 shutdownNow 获取不到任务，空闲下来，就会被销毁掉。这也就是 shutdown 会执行完队列里面任务的原因

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 当状态为 STOP 状态，或者 SHUTDOWN 状态且队列为空，获取不到任务
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
    ...
}
```

### 饱和策略（拒绝策略）

当线程池关闭了或在关闭状态中，或者任务太多的时候，线程池会拒绝任务提交，这时候为了程序的健壮性，把这种情况交给 RejectedExecutionHandler 处理。我们就看其中一种处理器，CallerRunsPolicy(交给原线程执行)

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public CallerRunsPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            // 这里让原线程直接调用 run 方法，就是让原线程执行
            r.run();
        }
    }
}
```

## jdk 中预定义的线程池

jdk 中预定义了一些线程池供开发者使用，通过调用 Executors 的 static 方法创建预定义的线程池

- FixedThreadPool 固定线程数，适用于负载较重的服务器，使用了无界队列
- SingleThreadExecutor 单线程，适用于顺序的执行任务，使用了无界队列
- CachedThreadPool 会根据需要创建新线程，适用很多短期的任务，使用 SynchronousQueue
- WorkStealingPool 基于 ForkJoinPool
- ScheduledThreadPoolExecutor

我们稍微看一下它是怎么创建，可以发现，大部分都是使用 ThreadPoolExecutor 这个线程池

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    public static ExecutorService newWorkStealingPool(int parallelism) {
        return new ForkJoinPool
            (parallelism,
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }

    public static ExecutorService newWorkStealingPool() {
        return new ForkJoinPool
            (Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }

    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }

    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

### ScheduledThreadPoolExecutor

ScheduledThreadPoolExecutor 继承了 ThreadPoolExecutor，还实现了 ScheduledExecutorService 接口

这个线程池适用定期执行任务。执行的线程建议捕捉异常，防止中断了定期执行。 ScheduledThreadPoolExecutor 具有 execute 和 sumbit 以外的方法

- schedule 可以延时执行
- scheduleAtFixedRate 提交固定时间间隔任务。两个开始的头之间是固定的
- scheduleWithFixedDelay 提交固定延时间隔任务。结束的尾巴和开始的头之间是固定的

## 合理配置线程池

多线程任务可以分为 计算密集型(CPU)、IO密集型，混合型 这几种类型。我们需要合理配置线程池，如果计算密集型的任务开启太多的线程，会让 CPU 上下文切换花费很多时间；如果 IO 密集型任务，配置太少的线程，IO 阻塞的时候线程被挂起，CPU 使用率就变低。所以需要考虑任务的类型来配置线程的多少

- 计算密集型
  - 例子：加密、正则、大数分解
  - 线程数应当适当的小，CPU 核心数+1。为什么+1，因为数据有可能要从磁盘加载到内存，这个称为页缺失
- IO 密集型
  - 例子：读取文件、数据库操作、网络资源
  - 线程数适当的大，CPU核心数*2

同时对阻塞队列选择有界队列，防止 OOM

总的来说线程池就聊到这里啦，一般建议用线程池的时候，自己指定线程池的参数，而不使用 Executors 去创建，因为开发人员需要明确的去定义自己的需求，还有另外一个原因是，Executors 创建的线程池是无界的，这有一定的风险。至于线程池底层是通过什么实现的呢，这里就不贴代码了，有兴趣可以翻翻代码，毕竟有了多线程基础后去看这些代码就会容易很多。底层实现还是绕不开 CAS，显示锁，AQS，代码中也经常可以看到通过位(bit)来做控制，这样的设计也可以多了解一下～