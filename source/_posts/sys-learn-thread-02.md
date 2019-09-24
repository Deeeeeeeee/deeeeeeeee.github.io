---
title: 【系统学习】【二】并发编程-常用并发工具类
categories:
  - java 学习
tags:
  - Java
  - thread
date: 2019-09-23 19:31:48
---

## 概要

最近天气开始转凉，所以今天来浅聊一下常用的几个并发工具类，然后请记住 *Doug Lea*  这个名字，当有人问起这个名字，就说我经常看他写的代码。并发工具类经常用的到，面试也经常会问，平时看源码时不时会蹦出来。所以今天*浅聊*的有：

- Fork-Join
- CountDownLatch 和 CyclicBarrier
- Semaphore
- Exchanger
- Callable/Future/FutureTask

<!-- more -->

## Fork-Join

ForkJoin 框架包括 ForkJoinPool、ForkTask、RecursiveTask 和 RecursiveAction，后面两个是 ForkTask 的抽象子类。作为浅聊，我们就来了解一下它们的基本思想，怎么使用和使用的场景

### Fork-Join 基本思想

Fork-Join 基本思想是分而治之，是不是很熟悉的名词，快排、归并，Hadoop 的 map/reduce 中都有这种思想。其实是很常见的一种思想，一句话来说，就是分成小任务，然后再合并起来

{% asset_img fork-join.png %}

规模为N的问题，当N<阈值直接解决；当N>阈值，将问题分解为K个小规模问题，这些问题相互独立，与原问题形式相同，最后将子问题的解合并得到原问题的解

### Fork-Join 工作密取

先来看一段 jdk 里面 ForkJoinPool 的描述

```java
/**
 * <p>A {@code ForkJoinPool} differs from other kinds of {@link
 * ExecutorService} mainly by virtue of employing
 * <em>work-stealing</em>: all threads in the pool attempt to find and
 * execute tasks submitted to the pool and/or created by other active
 * tasks (eventually blocking waiting for work if none exist). This
 * enables efficient processing when most tasks spawn other subtasks
 * (as do most {@code ForkJoinTask}s), as well as when many small
 * tasks are submitted to the pool from external clients.  Especially
 * when setting <em>asyncMode</em> to true in constructors, {@code
 * ForkJoinPool}s may also be appropriate for use with event-style
 * tasks that are never joined.
 */
```

意思是 ForkJoinPool 跟其他 ExecutorService 不同的是它采用了工作密取 work-stealing。工作密取是分而治之分割了每个任务之后，某个线程提前完成了任务，就会去其他线程偷取任务来完成，加快执行效率。同时，第一个分配的线程是从队列中的头部拿任务，当完成任务的线程去其他队列拿任务的时候是从尾部拿任务，所以这样就避免了竞争

### Fork-Join 使用

标准范式，就是主流的使用套路

```java
// 使用 ForkJoinPool
pool = new ForkJoinPool()
MyTask task = new ForkJoinTask()
pool.submit(task) // submit、execute 是非阻塞方法。pool.invoke(task) 是阻塞的
result = task.join()

// MyTask 实现 compute 方法
if (满足条件) {
  直接计算
  将结果提交
} else {
  拆分MyTask为两个或多个任务
  任务fork
  将结果提交
}
```

这里有两个 demo
- [计算数组的和](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/forkjoin/CalSumDemo.java)
- [列出文件夹下的所有 java 文件](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/forkjoin/ShowFileDemo.java)

```java
private static class MyTask extends RecursiveTask<Integer> {
    private static final int THRESHOLD = ARRAY_SIZE / 10;  // 阈值
    private int[] source;                       // 需要求和的数组
    private int left;
    private int right;

    public MyTask(int[] source, int left, int right) {
        this.source = source;
        this.left = left;
        this.right = right;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        if ((right - left) < THRESHOLD) {
//                SleepTool.sleep(1);
            for (int i = left; i <= right; i++) {
                sum += source[i];
            }
        } else {
            int mid = (left + right) / 2;
            MyTask leftTask = new MyTask(source, left, mid);
            MyTask rightTask = new MyTask(source, mid + 1, right);
            leftTask.fork();
            rightTask.fork();
            sum = leftTask.join() + rightTask.join();
        }
        return sum;
    }
}
```

## CountDownLatch 和 CyclicBarrier

我们先来看看这两个工具可以干一些什么事情
- CountDownLatch 是一个计数器，一组线程等待其他线程执行，然后再开始执行就可以使用 CountDownLatch。比如应用启动需要几个线程做完初始化工作，才开始运行
- CyclicBarrier 是一个栅栏，线程到达栅栏被阻塞，所有线程需要都到达栅栏位置，栅栏才打开，线程继续执行

这两个工具都可以协调线程，经常拿来比较，他们之间不同的是
- CountDownLatch 由第三方线程控制，放行条件 >= 线程数
- CyclicBarrier 由线程本身控制，放行条件 = 线程数
- 另外 CyclicBarrier(int parties, Runnable barrierAction) 栅栏开放，barrierAction 会执行

这里有两个demo
- [CountDownLatch 等待初始化线程完成](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/tool/CountDownLatchDemo.java)
- [CyclicBarrier 等待计算线程计算完毕](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/tool/CyclicBarrierDemo.java)

```java
public class CountDownLatchDemo {
    private static final CountDownLatch startSignal = new CountDownLatch(1);
    private static final CountDownLatch latch = new CountDownLatch(5);

    private static class MyTread extends Thread {
        @Override
        public void run() {
            try {
                startSignal.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("init thread " + this.getName() + " do init work");
            SleepTool.sleep(500);
            latch.countDown();
            System.out.println("init thread" + this.getName() + " continue work");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 5; i++) {
            MyTread myTread = new MyTread();
            myTread.start();
        }
        System.out.println("main thread let init thread work");
        // 开始信号
        startSignal.countDown();
        System.out.println("main thread wait init work finished");
        // 等待初始化完成
        latch.await();

        System.out.println("main thread continue work");
    }
}
```

## Semaphore 信号量

信号量：控制同时访问某个特定资源的线程数量，保证资源合理使用，用于流量控制

[这里模拟一个数据库连接](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/tool/semaphore/DBPool.java)

```java
public class DBPool {
    private final LinkedList<SqlConnection> pool = new LinkedList<>();
    // 可以使用连接
    private Semaphore canUsed = new Semaphore(20);
    // 已经使用连接
    private Semaphore hasUsed = new Semaphore(0);

    public DBPool() {
        for (int i = 0; i < 20; i++) {
            pool.addLast(new SqlConnection());
        }
    }

    public SqlConnection takeConn() throws InterruptedException {
        canUsed.acquire();
        SqlConnection conn;
        synchronized (pool) {
            conn = pool.removeFirst();
        }
        hasUsed.release();
        return conn;
    }

    public void releaseConn(SqlConnection conn) throws InterruptedException {
        if (conn == null) {
            return;
        }
        hasUsed.acquire();
        synchronized (pool) {
            pool.addLast(conn);
        }
        canUsed.release();
    }

    public static void main(String[] args) {
        DBPool pool = new DBPool();
        for (int i = 0; i < 50; i++) {
            new Thread(() -> {
                long start = System.currentTimeMillis();
                try {
                    SqlConnection connect = pool.takeConn();
                    System.out.println("Thread_"+Thread.currentThread().getId()
                            +"_获取数据库连接共耗时【"+(System.currentTimeMillis()-start)+"】ms.");
                    connect.createStatement();
                    connect.commit();
                    System.out.println("查询数据完成，归还连接！");
                    pool.releaseConn(connect);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

## Exchanger

交换线程数据的工具。两个线程间的数据交换，使用场景，只有一个生产者和一个消费者

由于这个只运行两个线程间进行交换，使用受到了不少限制，不是很实用，这里就不写 demo 了

##  Callable/Future/FutureTask

FutureTask 包装 Callable 常用来启动线程，跟 Runnable 不同的是，它可以有返回值。在线程池中经常可以看到

我们先来看看 FutureTask，可以看到 FutureTask 其实实现了 Runnable 接口

{% asset_img futuretask.png %}

再来看看 FutureTask 实现的 run 方法，可以看到执行了 Callable 的 call 方法

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

Future 的几个方法平时也需要注意一下
- isDone 正常结束、异常结束、正常取消，都返回 true
- isCancelled 任务完成前被取消，返回 true
- cancel(boolean) 尝试终止任务。注意中断是协作式的
  - 任务还未开始，返回 false
  - 任务开始，cancel(true)，会尝试中断在运行的任务。中断成功返回 true
  - 任务已经开始，cancel(false)，不会尝试中断在运行的任务
  - 任务已经结束，返回 false

好了，今天就聊到这里了，其实已经聊了一天了...