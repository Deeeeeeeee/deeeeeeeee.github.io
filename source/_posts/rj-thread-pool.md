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
execute 方法做的是添加工人或将任务添加到工作队列中。其中的逻辑如下：
- 如果工人数量小于核心线程数量，则添加工人，并将任务作为初始任务给这个工人
- 否则将任务添加到工作队列中（workQueue.offer(command)），并做二次校验（校验是否应该添加工人）
- 尝试添加核心线程数以外的工人，如果仍失败，则拒绝本次任务的提交

看到这里，是不是觉得几行代码就将核心工程完成了！那么，任务交给谁去完成了呢？这就需要看一下内部类 Worker 了，所以**真正干活的是工人**。

## 工人 Worker
首先看一下工人的日常生活

{% note info %}
此处应有动画 \[捂脸\]，然而我的前端技术限制了我的想法。总之，工人就是不断地从任务队列中获取任务，然后不断地完成任务。
{% endnote %}

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    ...
    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    public void lock()        { acquire(1); }
    public void unlock()      { release(1); }
    ...
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
} 

```

由于想简单说说这个 Worker 的日常，所以只截取其中一段代码。
- 首先，worker 有两个重要的变量，thread 和 firstTask。
- Worker 构造函数中，可以看出，thread 还是 worker 自身，firstTask 是构造时候赋值的，表示第一个任务。
- 除了构造方法，还有一个方法是 run 方法，run 方法调用了 runWorker 方法。
- runWorker 方法，就是在 worker 线程中，不停地 getTask，然后去调用获取到的任务的 run 方法。也就相当于不停地干着获取到的任务。

那么 worker 又是什么时候开始工作的呢？也就是什么时候开始执行自身的 run 方法的？这就要看上面 excute 方法里面调用的 addWorker 方法。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
                firstTask == null &&
                ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

addWorker 方法只干两件事情
- 在条件允许的情况下，添加工人
- 让工人开始干活（t.start()）

好了，到这里，主要的流程都明白了，当然，还有异常处理等流程还没有看。那么有几个问题，如果当多个线程同时去获取一个任务的时候怎么处理？怎么保证线程池中的工人数量跟预期的是一样的？（比如，你认为有 10 个工人，实际可能有1000个线程）当任务失败了，是怎么处理工人的呢？...

## 防止争抢任务
当多个线程同时获取一个任务的时候...未完待续...

<!-- ThreadPool 的 defaultThreadFactory 创建的线程是同一个 ThreadGroup 的，线程优先级为 NORM_PRIORITY 的非守护线程

-->