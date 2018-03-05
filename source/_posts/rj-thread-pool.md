---
title: 阅读源码系列--线程池 ThreadPoolExecutor
categories:
  - 阅读源码
tags:
  - jdk
  - threadPool
date: 2018-03-04 08:56:23
---

{% note success %}
通过使用标准化的，广泛测试过的并发构建模块，可以消除很多潜在的线程编程风险，诸如死锁、饥饿、竞争条件(race conditions)和过度的上下文切换。

[*jdk 说明文档*](https://docs.oracle.com/javase/8/docs/technotes/guides/concurrency/overview.html)
{% endnote %}
看了一下 ThreadPoolExecutor 源码的实现，觉得真的是赏心悦目。**写的很漂亮**，至少在我眼里是很漂亮的。

<!-- more -->

ThreadPoolExecutor 是并发实用工具(utilities)中 Task scheduling framework 里面的默认的线程池实现，具有灵活性和可扩展性。并发实用工具还有 Fork/join framework、Concurrent collections、Atomic varibles、Synchronizers、Locks 和 Nanosecond-granularity timing。有时间再去翻翻其他相关的实现，啊哈哈哈。
我阅读源码的 jdk 版本为 **1.8.0_131**，**强烈建议**先读 [*api文档*](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)，再阅读 jdk 的代码，会让你的思路变得清晰。

# 类图

{% asset_img uml.png %}

简单基本的关系图，继承关系非常清晰。其中 Future 属于任务提交后的异步结果，Executors 是创建 ExecutorService、Callable 等的工厂，RejectedExecutionException 是任务不能被接收时抛出的异常。
接下来简单看一下接口和抽象类。
## Executor
```java
public interface Executor {
    void execute(Runnable command);
}
```
Executor 接口将任务提交和线程使用的细节、调度解耦，也是该并发框架的核心，只有一个 void execute(Runnable command) 方法，通常负责创建线程。

## ExecutorService
```java
public interface ExecutorService extends Executor {
    // 终止线程池
    void shutdown();
    
    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    // 提交任务
    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);
    // 集中处理任务
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
ExecutorService 更像是对 Executor 的补充，提供终止线程池、提交任务和集中处理任务。

## AbstractExecutorService
```java
public abstract class AbstractExecutorService implements ExecutorService {
    // 装饰任务
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    ...
}
```
AbstractExecutorService 除了实现 submit 和 invoke 相关的方法外，还新增了两个装饰任务的方法 newTaskFor。这两个方法在新增任务的时候会被调用。

# 简单使用
官方推荐使用 Executors 创建线程池，默认创建的是 ThreadPoolExecutor，当然也可以继承 ThreadPoolExecutor， 并使用自己定义的线程池。为了简便，以下实例直接从 [*api 文档*](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ExecutorService.html)  中搬运过来。

```java
 class NetworkService implements Runnable {
   private final ServerSocket serverSocket;
   private final ExecutorService pool;

   public NetworkService(int port, int poolSize)
       throws IOException {
     serverSocket = new ServerSocket(port);
     pool = Executors.newFixedThreadPool(poolSize);
   }

   public void run() { // run the service
     try {
       for (;;) {
         pool.execute(new Handler(serverSocket.accept()));
       }
     } catch (IOException ex) {
       pool.shutdown();
     }
   }
 }

 class Handler implements Runnable {
   private final Socket socket;
   Handler(Socket socket) { this.socket = socket; }
   public void run() {
     // read and service request on socket
   }
 }
```
终止线程池的方法
```java
 void shutdownAndAwaitTermination(ExecutorService pool) {
   pool.shutdown(); // Disable new tasks from being submitted
   try {
     // Wait a while for existing tasks to terminate
     if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
       pool.shutdownNow(); // Cancel currently executing tasks
       // Wait a while for tasks to respond to being cancelled
       if (!pool.awaitTermination(60, TimeUnit.SECONDS))
           System.err.println("Pool did not terminate");
     }
   } catch (InterruptedException ie) {
     // (Re-)Cancel if current thread also interrupted
     pool.shutdownNow();
     // Preserve interrupt status
     Thread.currentThread().interrupt();
   }
 }
```
使用起来很简单，只要将需要执行的任务提交给线程池即可，线程管理、调度等都会帮你搞定。但是，只有这样的功能足够优秀么？

首先有个概念，**线程池“雇佣了”多名工人去执行任务**。工人指的就是线程池中的线程，任务指的是原本需要创建的线程（现在交给了线程池执行）。所以节省了线程频繁创建和销毁的开销，并且可以控制线程占用资源。
ThreadPoolExecutor 还提供核心工人数的设置、预创建工人、创建新工人的 factory、工人存活时间设置、数据结构的设置、四种异常处理策略、hook、清除任务等功能。具有不错的灵活性和健壮性。

说了这么多，最想了解的还是对线程管理的那一块，很明显，就是 execute 方法了。其余的设计放到后面再进行阅读。所以，接下来阅读的是 execute 方法。

# execute 方法
{% note success %}
execute 方法负责创建线程和安排任务，可以说是一个调度者，考虑什么时候要加工人，安排任务需要注意什么。也是线程管理的关键。
{% endnote %}

## 代码
```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

未完待续...

<!-- ThreadPool 的 defaultThreadFactory 创建的线程是同一个 ThreadGroup 的，线程优先级为 NORM_PRIORITY 的非守护线程

core max
pre 预处理
创建新线程 factory
keep-alive
队列（数据结构）
异常处理，警察
aop(hook)
处理队列,remove 等

最重要的是线程处理，即 execute 方法
但是 execute 方法又交给了 woker 去处理请求。每一个 worker 就是一个线程，

问题 worker 什么时候结束？
Worker 是什么，做了什么处理？
怎么保证性能的高效？
-->