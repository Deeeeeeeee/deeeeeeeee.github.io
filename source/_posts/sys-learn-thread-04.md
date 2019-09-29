---
title: 【系统学习】【四】并发编程-显示锁和AQS
categories:
  - java 学习
tags:
  - Java
  - thread
date: 2019-09-27 11:42:26
---

## 概要

我们除了可以使用 synchronized 内置锁外，jdk 还提供了 ReentrantLock 等显示锁，synchronized 方便使用，显示锁在某些方面更灵活。而显示锁的基础是 AQS，其实前几篇提到的几个工具类也是由 AQS 实现的，比如 CountDownLatch，CyclicBarrier 等，可以说 AQS 占有 jdk 并发包的半壁江山。所以今天我们就聊聊以下几个方面，并且看一点源码

- Lock 接口
- ReentrantLock 可重入锁
- ReadWriteLock接口 读写锁
- Condition 接口
- LockSupport 工具
- AQS(AbstractQueuedSynchronizer)
  - 模板方法设计模式
  - 数据结构：节点和同步队列
  - 独占式和共享式同步状态获取和释放
  - Condition 实现
  - 回看ReentrantLock、ReentrantReadWriteLock实现

<!-- more -->

## Lock 接口

{% asset_img lock.png %}

Lock 接口是显示锁的接口，包括以下几个方法
- lock/unlock 获取锁/释放锁
- tryLock 获取锁成功返回true，否则返回false。一般实现为非阻塞
- lockInterruptibly 可中断获取锁，即线程获取锁的过程中，收到中断信号，直接抛出 InterruptedException。而 lock 不能中断
- newCondition 创建一个条件，是用来实现 wait/notify 效果的

[lock 不会中断，lockInterruptibly 会中断获取锁的过程](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/lock/ReentrantLockTestDemo.java)

```java
  private static ReentrantLock lock = new ReentrantLock();

  private static class LockThread extends Thread {
      @Override
      public void run() {
          lock.lock();
          System.out.println("continue");
      }
  }

  private static class LockLockInterruptiblyThread extends Thread {
      @Override
      public void run() {
          try {
              lock.lockInterruptibly();
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  }

  public static void main(String[] args) throws InterruptedException {
      // 主线先拿到锁
      lock.lock();
      Thread.sleep(100);
      
      // 测试线程再拿锁的时候就被阻塞住了
//        LockThread thread = new LockThread();
      LockLockInterruptiblyThread thread = new LockLockInterruptiblyThread();
      thread.start();
      Thread.sleep(100);
      
      // 中断测试线程，LockThread 一直阻塞，而 LockLockInterruptiblyThread 中断了获取锁的过程
      thread.interrupt();
      System.out.println("finished");
  }
```

我们再拿 Lock 和 synchronized 比较

- synchronized 代码比较简洁，而且不容易出错
- 有以下几种情况可以使用 Lock
  - 获取锁可以被中断
  - 超时获取锁
  - 尝试获取锁(非阻塞)
  - 读多写少

## ReentrantLock

### 可重入

ReentrantLock 是可重入锁，即可以多次获取同一个锁，并且内部会计数。另外 synchronized 也是可重入锁

重入例子
- 比如线程调用了方法A拿了一次锁，在方法A里面调用了方法B拿了一次同一个锁。这时候线程拿了两次锁，当释放的时候，也需要释放两次
- 递归调用时拿了锁

### 公平锁和非公平锁

我们先来看一下 ReentrantLock 的一个构造函数

```java
    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

他们的区别

- 公平锁：如果在时间上，先对锁进行获取的请求，一定先被满足。非公平锁则不能满足，但效率更高
- 公平锁慢是因为挂起的线程解除挂起需要一定的时间，如果这时候让新来的线程先去拿锁，就节约了这一部分时间

这个说起来有点绕，等会看看源码就知道怎么回事了

## ReadWriteLock接口 读写锁

ReentrantLock、synchronized 是排他锁，即同一时刻只有一个锁。有时候不需要很严格的限制，希望可以有多个线程同时拥有一个锁，这时候就可以使用 ReadWriteLock。它的特点是**读写锁，同一时刻允许多个读线程访问，但只有一个写线程访问，写的时候其他读和写都阻塞**，适合读多写少的情况使用

[这个例子中使用读写锁跟synchronized相比耗时差不多是十分之一](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/lock/rw)

```java
public class GoodsRwOps implements GoodsOps {
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private Goods goods;

    public GoodsRwOps(Goods goods) {
        this.goods = goods;
    }

    // 获取价格
    public double getPrice() {
        lock.readLock().lock();
        try {
            SleepTool.sleep(1);
            return this.goods.getPrice();
        } finally {
            lock.readLock().unlock();
        }
    }

    // 设置价格
    public void addPrice(double newPrice) {
        lock.writeLock().lock();
        try {
            SleepTool.sleep(5);
            this.goods.setPrice(this.goods.getPrice() + newPrice);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

## Condition 接口

我们都知道 synchronized 有一个等待通知功能，Condition 就是为显示锁打造的等待通知，而且更灵活一点

{% asset_img condition-interface.png %}

跟之前聊的 wait/notify 一样，通过 Condition 也可以实现等待通知功能。不一样的是，一个显示锁可以创建多个 Condition，而且推荐使用 signal 方法，而不是 signalAll 方法，因为 Condition 可以精准唤醒该条件的等待线程。而 wait/notify 由于不能确定唤醒的线程是否目标等待线程，所以推荐使用 notifyAll 方法进行唤醒

这里还有一个地方要注意的**跟wait一样，在condition.await之前需要拿锁，执行await之后，会自动释放这个锁，await执行完后又会自动获取到锁**

[完整例子](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/lock/ConditionDemo.java)

```java
public static class WaitClazz {
    // 是否需要等待的条件
    private boolean waitFlag = true;
    private ReentrantLock lock = new ReentrantLock();
    private Condition startCd = lock.newCondition();
    private Condition endCd = lock.newCondition();

    public void waitSomething(boolean isStartSignal) throws InterruptedException {
        lock.lock();
        try {
            Condition condition = isStartSignal ? startCd : endCd;
            while (waitFlag) {
                condition.await();
            }
            System.out.println(String.format("接收到【%s】通知，继续干活", isStartSignal ? "开始" : "结束"));
        } finally {
            lock.unlock();
        }
    }

    public void notifySomething(boolean isStartSignal) {
        lock.lock();
        try {
            waitFlag = false;
            Condition condition = isStartSignal ? startCd : endCd;
            condition.signal();
            System.out.println(String.format("发出【%s】通知信号", isStartSignal ? "开始" : "结束"));
        } finally {
            lock.unlock();
        }
    }
}
```

## LockSupport 工具

LockSupport 是通过本地(native)方法，将线程阻塞和唤醒的工具，这个工具在 AQS，线程等多个并发工具类都有使用到，是构建同步组件的基础工具

{% asset_img lockSupport.png %}

- park 是阻塞线程
- unpark 是唤醒线程

## AQS(AbstractQueuedSynchronizer)

AQS 是 AbstractQueuedSynchronizer 的缩写，即抽象的同步器(队列实现的)，是很多同步组件的基础，所谓占据并发工具的半壁江山。我们先来看看它有什么功能，然后使用 AQS 实现一个自己的显示锁，然后再进一步分析 ReentrantLock 是怎么实现的，最后在看看一下其他并发工具类是如何通过 AQS 实现的。我们将会阅读不少代码

### 模板方法设计模式

ReentrantLock 是通过 AQS 实现的，在我们实现之前，我们得先了解一下，我们需要实现方法。而 AQS 使用了模板方法设计模式，简单的说父类给定了一个流程，子类只需要实现各自不同的部分流程，本篇就不详细说这个模式了，装不下这么多的内容 =。=

AQS 的方法大致可以分为以下几类

- 模板方法
  - 独占式获取锁：accquire/accquireInterruptibly/tryAcquireNanos
  - 共享式获取锁：acquireShared/acquireSharedInterruptibly/tryAcquireSharedNanos
  - 独占式释放锁：release
  - 共享式释放锁：releaseShared
- 需要子类实现的方法
  - 独占式： tryAcquire/tryRelease
  - 共享式： tryAcquireShared/tryReleaseShared
  - 这个同步器是否独占式的：isHeldExclusively
- 同步状态 state 相关的方法：getState/setState/compareAndSetState

所谓的独占式，就是排它的意思，一个时刻只有一个线程可以拥有这个锁；共享式就是读写锁中的读锁

至于需要子类实现的方法，是因为 AQS 把这些方法留给子类来实现，然后在模板方法中会调用，比如 acquire 方法会调用 tryAcquire 方法

```java
// 这里 acquire 调用了 tryAcquire 方法，只是 AQS 本身没有实现 tryAcquire 方法
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

好了，我们接下来实现一个独占式的显示锁，所以我们需要实现 tryAcquire/tryRelease/isHeldExclusively 这几个方法。稍微看一下 ReentrantLock，发现 AQS 的实现是交给 Sync 这个内部类去实现的，我们也这么干

```java
// 可以看到，AQS 的实现交给了 Sync
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;

    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {
      ...
```

然后再观察一下发现 ReentrantLock 是根据 state 状态进行判断的，我们就也这么干，state == 1 表示拿到了锁，state == 0 表示没有拿到锁，先简单的实现一下。然后还有一个 Condition，这个直接调用 new ConditionObject() 就好了，所以最后，我们的实现如下

```java
public class SelfLock implements Lock {
    private final Sync sync = new Sync();

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            if (getState()== 1) {
                setState(0);
                setExclusiveOwnerThread(null);
                return true;
            }
            return false;
        }

        @Override
        protected boolean isHeldExclusively() {
            return true;
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Deserializes properly
        private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0);    // reset to unlocked state
        }
    }

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }

    public boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }
    
    public boolean isBlocked() {
        return sync.isHeldExclusively();
    }
}
```

我们的显示锁实现还不到100行，是不是觉得挺容易的，其实真的也没有那么复杂，获取锁的时候，用 CAS 把 state 设置成 1，并且 setExclusiveOwnerThread 把当前线程设置为独占锁的线程，释放锁的时候刚好相反。这里还有个地方需要注意，释放锁的时候，因为只有一个线程进行释放锁，就不需要 CAS 了

[测试可以在这里测试](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/lock/selflock/SelfLockTestDemo.java)，或者将之前 Condition 例子中的 ReentrantLock 改成我们的 SelfLock

[这里还有一个类似于CountDownLatch的实现](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/lock/selflock/BooleanLatch.java) 这个是共享式的

### 数据结构：节点和同步队列

我们实现了一把显示锁，觉得有点兴奋，觉得有点简单，但肯定有点迷惑，AQS 到底是怎么做到的呢。我们接下来看看它是怎么实现的，一般我们做需求，首先要知道这个需求的目标是什么，然后再思考实现的思路，最后开始实现其中的细节。我们也按照这样的思路来看 AQS 的源码

- AQS 的功能
  - 我希望 AQS 可以解决线程竞争问题
  - 我希望线程拿锁的时候如果失败就阻塞，等到有机会的时候可以自动拿到锁，然后继续干活
  - 我希望释放锁的时候，可以自动通知到阻塞的线程去拿锁
  - 我还希望有超时功能
- 实现思路
  - 解决竞争问题可以利用 CAS 来解决，如果 CAS 失败，就表明竞争失败。我们需要引入 state，用来 CAS
  - 如果竞争失败，我们要把线程包装起来，包装成 Node，放到 FIFO(先进先出)队列当中，然后开始自旋，当 Node 前一个节点为 head 节点并且 CAS 成功，则退出自旋，将当前 Node 设置为 head 节点，继续干活
  - 改变 state 状态，使 head 节点下一个自旋中的线程 CAS 可以成功
  - 超时的实现在原来的实现上使用之前聊的 [等待超时范式](https://deeeeeeeee.github.io/post/sys-learn-thread-01) 实现

带着这个思路我们可以得到下面这幅图，这个只是大概的一个流程，还有一些 waitState 和一些细节没有在这里体现，因为我们总是希望从简单或简洁开始

{% asset_img AQS-state.png %}

然后我们先来看看内部类 Node 的代码

```java
static final class Node {
    // 标记共享式和独占式节点
    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    /** watiStatus 状态 */
    // 表示线程因为等待超时或中断而取消，需要从同步队列中移除
    static final int CANCELLED =  1;
    // 表示后续的线程需要 unparking(唤醒)，即当前线程释放锁或取消的时候去唤醒下一个线程
    static final int SIGNAL    = -1;
    // 表示线程正在等待 condition 通知，处于等待队列中
    static final int CONDITION = -2;
    // 表示共享式释放锁时，无条件地向下传播状态
    static final int PROPAGATE = -3;

    // 普通的节点默认是 0，condition 节点默认是 CONDITION(-2)，非负就不需要 signal
    volatile int waitStatus;

    // 前驱节点
    volatile Node prev;
    // 后续节点
    volatile Node next;

    // 被包装的线程
    volatile Thread thread;

    // condition 中下一个等待中的节点，或者是标记共享式节点的 SHARED
    Node nextWaiter;

    // 等待的节点是否共享式的
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 获取前驱节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

容易观察到 Node 有几个状态 waitStatus
- CANCELLED 表示线程因为等待超时或中断而取消，需要从同步队列中移除
- SIGNAL 表示后续的线程需要 unparking(唤醒)，即当前线程释放锁或取消的时候去唤醒下一个线程
- CONDITION 表示线程正在等待 condition 通知，处于等待队列中
- PROPAGATE 表示共享式释放锁时，无条件地向下传播状态
- 0 为普通节点的默认值，waitStatus 非负就不需要 signal

然后看看 AQS 的字段

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    // 首尾节点
    private transient volatile Node head;
    private transient volatile Node tail;

    // AQS 状态，由子类定义 state 状态
    private volatile int state;

    // static 字段，用来支持 CAS 操作。offset 表示字段在对象中的偏移量，这个跟底层的内存地址排列有关
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;
}
```

到这里，我们可以确定 AQS 使用的是**双向队列**数据结构，这个大家都知道原理是什么样的，就不解释了。然后节点包装了线程，同步队列先进先出(FIFO)，当线程去拿锁拿不到的时候，就需要进入同步队列

### 独占式和共享式同步状态获取和释放

接下来我们该看多一点源码了。我们先来看独占式锁的获取和释放，方法有以下几个

```java
    // 先尝试获取锁，如果失败则加入同步队列，并开始自旋获取锁
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    // 如果线程是中断状态，抛出 InterruptedException；否则尝试获取锁，如果失败，则加入同步队列，并开始自旋获取锁
    // 当收到中断信号，抛出 InterruptedException
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

    // 如果线程是中断状态，抛出 InterruptedException；否则尝试获取锁，如果失败，则加入同步队列，并开始自旋获取锁
    // 当收到中断信号，抛出 InterruptedException
    // 当等待超时的时候，返回 false
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

    // 尝试释放锁，如果释放成功，waitStatus 不是 0 则唤醒后驱节点
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

可以了解到，三个获取锁的实现其实都是差不多的，其中 tryAcquire 调用的是子类的方法，子类方法需要使用 CAS 操作，保证原子性。然后我们再看一下 addWaiter 和 acquireQueued 是怎么包装线程为 Node，接着怎么开始自旋的

```java
    // 将当前线程包装成 Node，然后用 CAS 将节点添加到队列尾部
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

    // 自旋将 Node 添加进队列
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

    // 自旋检查前驱节点是否释放锁，如果是设置 head 节点
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    // 线程如果收到了中断信息号，则返回 true. 这个体现的是 java 的中断是协作式的
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

看到这里，是不是觉得豁然开朗了，只要前驱节点释放了锁，当前节点就可以拿到锁了。其实这时候就可以看看 CLH 锁了，可以看看 CLH 为了解决什么问题，这是 93 年的那篇[论文](ftp://ftp.cs.washington.edu/tr/1993/02/UW-CSE-93-02-02.pdf)

{% asset_img AQS-node.png %}

我们继续看一下可中断的获取锁和超时获取锁

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 只有这一句不同，抛出了异常，直接退出自旋，交给原来的线程处理
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 如果了解等待超时范式的话，就很容易知道这是怎么实现的了
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    // 超时时间
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 更新超时时间
            nanosTimeout = deadline - System.nanoTime();
            // 判断超时
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

看到这里，基本上就看完了，我们就知道了，子类在 tryAcquire 和 tryRelease 管理好 state 就能实现好锁了

### Condition 实现

AQS 除了 Node 内部类，还有一个 ConditionObject 内部类

```java
public class ConditionObject implements Condition, java.io.Serializable {
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;
    ...
}
```

这个也是一个队列，我们称为等待队列跟 AQS 共用 Node 数据结构。同样的我们通过思考实现思路，不难得出下面这个流程

{% asset_img condition.png %}

我们来看看怎么进入等待队列，怎么移入同步队列的。最主要的两个方法 await 和 signal。先开 await

```java
// 等待唤醒并且不能中断
public final void awaitUninterruptibly() {
    // 进入等待队列
    Node node = addConditionWaiter();
    // 释放锁
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    // 如果节点不在同步队列中，循环
    while (!isOnSyncQueue(node)) {
        // 挂起，等待被唤醒
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true;
    }
    // 自旋竞争锁
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}

// 可以中断的等待
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        // 当有其他线程调用unpark、该线程被中断、park方法异常返回，都会被唤醒
        LockSupport.park(this);
        // 如果线程被中断，退出循环，进入同步队列
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}

// 将节点加入等待队列中
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // waitStatus 为 CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

我们发现有判断 node 是否在同步队列中，但是怎么移入到同步队列中的呢。我们再看看 signal

```java
// 取第一个等待节点进行操作
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

// 移出等待队列，firstWaiter 设置成下一个，由于节点可能取消了，所以需要循环
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
                (first = firstWaiter) != null);
}

// 将 node 转换成同步队列的节点
final boolean transferForSignal(Node node) {
    /*
        * If cannot change waitStatus, the node has been cancelled.
        */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    // 将 node 放入同步队列
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

可以看到，signal 将节点移除了等待队列，并把它放到了同步队列当中，这样等待的 node 就可以开始自旋拿锁了。这样就实现了等待通知的功能

### 回看ReentrantLock、ReentrantReadWriteLock实现

我们已经知道了 AQS 的原理和它的实现了，那么我们可以回头看看 ReentrantLock 是怎么实现的。我们上面实现的 SelfLock 跟 ReentrantLock 相比，其实是不可重入的，而且也是非公平的。我们同样可以思考一下怎么实现这两个功能

- 由于最关键的 state 管理是交给我们的，所以我们判断是同一个线程来拿锁的时候，就让它能拿成功，并且用 state 计数
- 公平锁的拿锁的时候，要判断当前线程是否是 head 的下一个节点，如果不是，就不能拿锁

我们来看看

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    // 非公平锁先尝试一次占有锁，这是一种贪婪的策略，也是比公平锁快的原因
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}

static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 公平锁比非公平锁多了一个限制，要先判断是否是 head 的下一个节点。hasQueuedPredecessors 是专门拿来实现公平机制的方法
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // CAS 拿锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 判断当前线程是否独占线程
    else if (current == getExclusiveOwnerThread()) {
        // 将 state 计数增加
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这么一看，如果同一线程多次拿锁，state 增加就好了，只要在释放的时候，再将 state 减去就行。释放锁的代码就不贴出来了

我们再来看一下 ReentrantReadWriteLock，先来看一下它的字段

```java
public class ReentrantReadWriteLock
        implements ReadWriteLock, java.io.Serializable {
    /** Inner class providing readlock */
    private final ReentrantReadWriteLock.ReadLock readerLock;
    /** Inner class providing writelock */
    private final ReentrantReadWriteLock.WriteLock writerLock;
    /** Performs all synchronization mechanics */
    final Sync sync;
    ...
}
```

可以看到有两个锁，读锁和写锁，读锁使用的是共享式的，写锁是独占式的。其实实现跟独占式的差不多，只是 state 计数上不同，它将 state 分为两部分，低 16 位给独占式计数，高 16 位给共享式计数。但是共享式计数只是记线程数，具体的重入次数给到 ThreadLocal 去维护

由于时间关系，本篇就先到这里了，其实还有很多可以聊的。我们还没聊 AQS 共享式是怎么实现的，Latch 门闩，栅栏是怎么通过 AQS 实现的，这些就让大家去仔细探索了