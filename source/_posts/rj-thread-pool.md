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
我阅读源码的 jdk 版本为 1.8.0_131，**强烈建议**先读 [*api文档*](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html)，再阅读 jdk 的代码，会让你的思路变得清晰。

# 类图

{% asset_img uml.png %}

简单基本的关系图，未完待续...

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