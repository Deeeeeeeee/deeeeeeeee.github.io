---
title: 【系统学习】【五】并发编程-并发容器
categories:
  - java 学习
tags:
  - Java
  - thread
date: 2019-10-07 14:34:04
---

## 概要

容器，这个概念在我刚开始接触的时候，其实很模糊，到处都可以看到容器这个概念。到现在我的理解起来就很简单了，就是一个装载空间，每个容器有它一定的功能，可进可出，就是 container。像并发容器，就像 ConcurrentHashMap，ConcurrentSkipArrayList，LinkedBlockingQueue 等，可以装载 java 对象，提供线程安全的一些功能。由于目前我常用的 jdk 是 8 的版本，所以这一次就聊 jdk8 里面的一些并发容器，至于网上经常拿 jdk7 和 jdk8 的并发容器进行比较，而且面试也可能问，但是官网上已经下载不到了，我也就懒得去找 jdk7 来对比了

所以我们这次要聊的有

- Hash 和 hashMap
- ConcurrentHashMap
- ConcurrentSkipListMap和ConcurrentSkipListSet
- ConcurrentLinkedQueue
- 写时复制容器
- 阻塞队列

<!-- more -->

## Hash 和 hashMap

我们先来聊一下哈希这个技术，因为接下来会使用到

- 概念：把任意一个输入通过一种算法（散列），变换成固定长度的输出，这个输出就是散列值
- 常用算法：
  - 直接取余数
  - md4,md5,sha 等摘要算法

### 位运算实现取模

这里插一个小知识，因为下面会使用到

当 b = 2^n 时，a & (b - 1) 想当于 a % b。比如 7 % 4 = 3 和 111 & 011 = 011。注意这里 b 只能是 2 的 n 次方数

### hashMap 中使用哈希算法

hashmap 的实现概念如下图，将 key 进行 hash，根据 hash 值分配到桶(bucket)

{% asset_img hashmap.png %}

- 放进(put) hashmap 的时候，根据 key 的 hash 值找到桶，封装 key, value 成 Node，添加进桶里（同一个桶里的 hash 值都是一样的）
- 查询(get) hashmap 的时候，根据 key 的 hash 值找到桶，在桶里查找 key.equals(k) 的 Node，返回 value
- 遍历 hashmap 的时候，遍历每一个桶和桶里面的 Node

jdk8 中的 hash 算法

```java
// 将 key 的 hashCode 和它本身高 16 位进行异或
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 然后 hash 取模 桶的个数，定位到桶的下标
tab[(n - 1) & hash]
```

hashmap 的主要实现思路其实就是上面说的那样，把 hash 值作为标识，然后根据标识把元素分散开来，就跟分库分表一样

### hashMap 非线程安全

并发情况下，可能会出现的情况。[简单的测试代码](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/container/HashMapTestDemo.java)

- 多个线程同时 put，元素可能会丢失

```java
private static Map<String, String> map = new HashMap<>();

public static void main(String[] args) throws InterruptedException {
    // 线程1 => t1
    Thread t1 = new Thread(new Runnable() {
        @Override
        public void run() {
            for (int i = 0; i < 999; i++) {
                map.put("thread1_key" + i, "thread1_value" + i);
            }
        }
    });
    // 线程2 => t2
    Thread t2 = new Thread(new Runnable() {
        @Override
        public void run() {
            for (int i = 0; i < 999; i++) {
                map.put("thread2_key" + i, "thread2_value" + i);
            }
        }
    });
    t1.start();
    t2.start();
    t1.join();
    t2.join();
    System.out.println(map.size());
}

// 输出结果 1938
```

形成的原因也很简单，多个线程同时往一个桶里面添加 Node 的时候，会出现覆盖的现象。比如下面这一段

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
      ...
    if ((e = p.next) == null) {
        // 两个线程都进入到这里，然后先后执行了这一句，p.next 就会被覆盖
        p.next = newNode(hash, key, value, null);
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        break;
    }
    ...
}
```

- 多个线程同时 put，get 获取的值可能为 null

这里就不贴代码了，可以看 [简单的测试代码](https://github.com/Deeeeeeeee/demos/blob/master/src/main/java/com/sealde/thread/container/HashMapTestDemo.java)

原因 resize 的时候，tab = newTable，此时还没有元素，get 得到了 null。其实还有其他原因

- jdk7 Entry链表可能形成环形数据结构，一旦形成环形数据结构，Entry的next节点永远不为空，就会产生死循环获取Entry

这个网上有一大堆分析，这里就不分析了。简单的说就是因为 jdk7 是头部新增数据的，即 bucket -> newOne -> oldOne，java8 解决了这个问题，即 bucket -> oldOne -> newOne。两个线程同时做扩容操作，因为头插法很可能直接将链表反转(jdk7 的 bucket 是链表)，当一个线程刚确认好 当前节点和一个节点时，另一个线程将链表反转了，这个线程继续执行，就会出现环，即 k1 -> k2，k2 -> k1。形成环之后，map.get(K) 时会遍历链表，由于链表形成环了，遍历就死循环了

## ConcurrentHashMap

ConcurrentHashMap 和 HashMap 的思路差不多，只是在 put，remove，扩容等操作的时候，加了锁或者使用 CAS 操作，保证在并发情况下的安全性

### ConcurrentHashMap jdk7 和 jdk8 的区别

这里聊一下主要区别

- 取消了 segment，直接使用一个 node 数组，锁的粒度更小，减少并发冲突的概率
- 链表+红黑树，纯链表O(n)，红黑树O(logn)，性能提升很大

[这里有一篇讲它们之间区别的](https://www.jianshu.com/p/933289f27270)

### ConcurrentHashMap 数据结构和关键变量

ConcurrentHashMap 使用到的几个数据结构，Node, TreeNode, TreeBin。java8 桶(bucket) 可以是链表，也可以是红黑树，用来装载 key-value，当链表元素个数超过阈值 8，将转换成红黑树；当红黑树元素个数少于阈值 6，将转换成链表

```java
// 链表使用的 Node 存储 key,value。 val 和 next 是 volatile 的，保证可见性
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}

// 红黑树使用 TreeNode 数据结构，注意继承了 Node
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
}

// 红黑树使用 TreeBin 代表树的 root
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // values for lockState
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock
}
```

其实还有两个数据结构 `ForwardingNode` 和 `ReservationNode`。以下是几个常量

```java
// 桶数组的最大个数
private static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认桶的个数
private static final int DEFAULT_CAPACITY = 16;
// 负载因子，当元素个数达到容量的 0.75 倍时，进行扩容。实际没有使用这个常量，而是通过 n-(n>>>2) 这样的位运算得出 0.75 倍的
private static final float LOAD_FACTOR = 0.75f;

// 树化阈值，链表上的 Node 达到 8 时，链表转换成红黑树
static final int TREEIFY_THRESHOLD = 8;
// 非树化阈值
static final int UNTREEIFY_THRESHOLD = 6;
//  最小树形化容量阈值：即 当哈希表中的容量 > 该值时，才允许树形化链表 （即 将链表 转换成红黑树）
// 否则，若桶内元素太多时，则直接扩容，而不是树形化
// 为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
static final int MIN_TREEIFY_CAPACITY = 64;

// 并发度，默认是 16。这个跟 CPU cache 命令率有关
private static final int MIN_TRANSFER_STRIDE = 16;
private static int RESIZE_STAMP_BITS = 16;
// help resize 最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
// 记录 sizeCtl 中的偏移量
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

/*
    * Node 里面 hash 字段的值
    */
static final int MOVED     = -1; // 表示正在扩容的节点
static final int TREEBIN   = -2; // 表示红黑树
static final int RESERVED  = -3; // 表示 ReservationNode，貌似是一种临时的节点
static final int HASH_BITS = 0x7fffffff; // 正常节点的 hash
```

变量

```java
// 数组容器，就是桶。java8 是懒初始化的
transient volatile Node<K,V>[] table;
// 下一个要使用的 table，只有 resize 的时候为 null
private transient volatile Node<K,V>[] nextTable;
// 统计元素数量，CAS 操作这个字段
private transient volatile long baseCount;

// 容器初始化和扩容控制单位。负数时，表示在初始化或者扩容。-1表示正在初始化，-n表示有n-1个线程正在进行扩容
// 0表示还未初始化
// 大于0表示初始化或者下一次扩容的阈值
private transient volatile int sizeCtl;
private transient volatile int transferIndex;
private transient volatile int cellsBusy;

// baseCount CAS 失败会使用该字段
private transient volatile CounterCell[] counterCells;
```

### ConcurrentHashMap 构造方法

跟 jdk1.7 不同，jdk1.8 的构造方法是懒初始化的，等具体使用到某个桶的时候才做初始化工作。而 jdk1.7 有 segment 概念，构造方法会初始化好 segment 数组

```java
// 默认容量16
public ConcurrentHashMap() {
}

// 初始化 sizeCtl 变量，表示下次扩容到多大
// tableSizeFor 保证容量为 2^n
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                MAXIMUM_CAPACITY :
                tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

// 可以看出，构造方法没有对 table 进行初始化，只是赋值了 sizeCtl 变量
public ConcurrentHashMap(int initialCapacity,
                            float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

### ConcurrentHashMap put

put，如果 key 已经存在，覆盖并返回原来的值；如果 key 不存在，填上并返回 null

**这里桶称之为 bin，容器的意思**

通过 CAS 操作和 对首节点 synchronized 加锁，保证添加元素的时候，该 bin 下的操作是线程安全的


```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 将对象的hashCode再hash一遍
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 如果不对应的桶不存在，CAS 添加新 Node。这里是懒初始化要初始化的时候
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 该桶正在扩容，帮忙扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 对首节点加锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    // 链表添加元素
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                break;
                            }
                        }
                    }
                    // 红黑树添加元素
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 如果 bin 数量大于树化阈值，进行树化操作
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 计数加 1
    addCount(1L, binCount);
    return null;
}
```

### ConcurrentHashMap get

get 方法是没有加锁的，但是跟 HashMap 不同的是，在扩容完成的时候才会 table = nextTable，所以不会出现获取到的值为 null 的情况

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 对 key 再 hash
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // hash 值相同，并且 key.equals(ek) 返回对应的值
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 首节点的 hash 值小于 0，表示非链表，比如红黑树
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 如果是链表，循环查找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

### ConcurrentHashMap 扩容和 size

putVal 的时候有一个 addCount 方法，里面判断元素个数是否达到 sizeCtl 阈值，如果达到阈值，则进行扩容操作。扩容操作是翻倍扩容，容量为原来的 2 倍

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 如果 baseCount CAS 失败，则会使用 CounterCell 统计
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
            // CounterCell CAS 统计
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 自旋 CAS 统计
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 判断是否达到阈值 sizeCtl，如果达到阈值，进行扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2))
                // 具体的扩容操作
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

size 方法里面主要是 sumCount 方法，这里是没有加锁的。在并发情况下，size 方法得出的结果不一定是精确的，是一个估值

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    // 统计 baseCount 的数据和 CounterCell 里面的数据
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

弱一致性，get、size 等方法没有加锁，会出现线程读取数据到一半，其他修改线程操作完毕，然后读取数据的线程继续执行，得到了旧的数据

### ConcurrentHashMap 面试常见问题

- ConcurrentHashMap实现原理是怎么样的或者问ConcurrentHashMap如何在保证高并发下线程安全的同时实现了性能提升？
  - ConcurrentHashMap允许多个修改操作并发进行，其关键在于使用了锁分离技术。它使用了多个锁来控制对hash表的不同部分进行的修改。内部使用段(1.7 Segment 1.8 Node)来表示这些不同的部分，每个段其实就是一个小的
hash table，只要多个修改操作发生在不同的段上，它们就可以并发进行
- 在高并发下的情况下如何保证取得的元素是最新的？
  - 用于存储键值对数据的HashEntry，在设计上它的成员变量value等都是volatile类型的，这样就保证别的线程对value值的修改，get方法可以马上看到
- 如何在很短的时间内将大量数据插入到ConcurrentHashMap
  - 将大批量数据保存到map中有两个地方的消耗将会是比较大的：第一个是扩容操作，第二个是锁资源的争夺。第一个扩容的问题，主要还是要通过配置合理的容量大小和扩容因子，尽可能减少扩容事件的发生；第二个锁资源的争夺，在put方法中会使用synchonized对头节点进行加锁，而锁本身也是分等级的，因此我们的主要思路就是尽可能的避免锁等级。所以，针对第二点，我们可以将数据通过通过ConcurrentHashMap的spread方法进行预处理，这样我们可以将存在hash冲突的数据放在一个组里面，每个组都使用单线程进行put操作，这样的话可以保证锁仅停留在偏向锁这个级别，不会升级，从而提升效率

更多面试题会补充上来

## ConcurrentSkipListMap和ConcurrentSkipListSet

- 是TreeMap、TreeSet 的并发版本
- 通过 CAS 保证线程安全性
- 使用的技术是跳表
  - 典型的空间换时间技术
  - 随机将元素作为索引，查询的时候使用索引会快很多
  - 这里有一篇描述 [redis 中使用跳表的](https://zhuanlan.zhihu.com/p/23370124)
  - 时间复杂度快赶上红黑树

## ConcurrentLinkedQueue

- LinkedList 的并发版本
- 通过 CAS 保证线程安全性
- 无界非阻塞队列，底层是个链表
- add,offer 从尾部添加数据。peek(拿数据不移除)，poll(拿数据要移除)，从头部移除数据

```java
// 典型的自旋 CAS
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (p.casNext(null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

## 写时复制容器

- CopyOnWriteArrayList、CopyOnWriteArraySet
- 思路是要修改的时候，将数据拷贝一份进行修改，然后修改引用
- 只能保证最终一致性，因为读取的时候可能读取到旧的数组
- 适用读多写少，白名单、黑名单、商品类目的更新

```java
// 写的时候使用显示锁
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // 拷贝数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}

// 修改引用
final void setArray(Object[] a) {
    array = a;
}
```

## 阻塞队列

- 核心思想：当队列满的时候，插入元素线程被阻塞，直到队列不满；当队列空的时候，获取元素线程被阻塞，直到队列不空
- 使用场景：生产者消费者模式。使用容器降低生产者和消费者之间的耦合
- 常用方法(插入/移除/检查)
    - 抛出异常类型：add/remove/element
    - 返回值类型：offer/poll/peek
    - 一直阻塞类型：put/take
    - 超时退出类型：offer(time)/poll(time)

### 具体的阻塞队列

- ArrayBlockingQueue 
  - 底层实现为数组、有界
  - 先进先出，需要设置初始大小
- LinkedBlockingQueue
  - 底层实现为链表、有界
  - 先进先出，可以不设置初始大小，默认 Integer.MAX_VALUE
  - 与 ArrayBlockingQueue 区别
    - 锁：ArryaBlockingQueue 只使用一个锁，LiknedBlockingQueue 使用两个锁
    - 数据结构上：ABQ 直接插入，LBQ 需要先转换成节点
- LinkedBlockingDeque
  - 链表、有界、双向
  - 双向获取和移除，工作密取有使用

- PriorityBlockingQueue
  - 优先级、无界
  - 默认情况下，按照自然顺序；实现 compareTo; 构造 Comparetor
- DelayQueue
  - 优先级队列、无界
  - 支持延时获取元素的阻塞队列，元素必需实现 Delayed 接口
  - 使用场景：缓存系统、订单到期、限时支付

- SynchronousQueue
  - 不存储元素
  - 每一个 put 等待 take

- LinkedTransferQueue
  - 链表、无界
  - 跟 LinkedBlockingQueue 区别
    - 多了 transfer()、tryTransfer()。先不进入队列，先查看有没有消费者，如果有，直接交给消费者
    - transfer 需要等待消费者消费才返回，tryTransfer 无论是否消费，直接返回

他们的底层实现都差不多，是通过显示锁和 Condition 实现的。我们来看其中的一个

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
    /** The capacity bound, or Integer.MAX_VALUE if none */
    private final int capacity;

    /** Current number of elements */
    private final AtomicInteger count = new AtomicInteger();

    /**
     * Head of linked list.
     * Invariant: head.item == null
     */
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     */
    private transient Node<E> last;

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();

    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        // 加锁
        putLock.lock();
        try {
            if (count.get() < capacity) {
                // 添加元素
                enqueue(node);
                // 计数加一
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    // 通知
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }

    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }
}
```