---
title: 生产问题-spring quartz 暂停
categories:
  - 生产问题
tags:
  - spring-quartz
  - Quartz
  - jstack
date: 2018-03-21 23:12:12
---

前几天生产上出现一个问题，spring 集成 quartz 无故停住了，重启又好了。导致一些功能受到限制，查询和沟通了一段时间才解决。

描述：表象，订单状态全部卡在了一个状态下，不再改变；日志，定时器执行一段时间后不再执行，而且是所有的定时器都不执行了。

<!-- more -->

# 开始思考

定时器卡住了的原因可能是：
- jvm 内存爆了，导致应用停止了（第一反应是这个～因为定时器都没有设置 concurrent = false）
- 线程阻塞了，或者在等待，死锁了

# 查询相关资料

{% note success %}
生产上使用的 spring 版本为 3.0.5
{% endnote %}

## spring 集成 quartz

在 [*spring 3.2.18 文档*](https://docs.spring.io/spring/docs/3.2.18.RELEASE/spring-framework-reference/htmlsingle/#scheduling) 看了半天才明白 spring 的定时器有两类，一类完全是 spring 代码里面编写的，一类是是集成了 [*Quartz*](http://quartz-scheduler.org/)。。。

### 常见配置
```java
<!-- 定时器注册工厂 -->
<bean id="SpringJobSchedulerFactoryBean" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
	<property name="triggers">
		<list>
			<ref bean="ftp-scan-trigger"/>
		</list>
	</property>
</bean>
<!-- 定时器配置 -->
<bean id="ftp-scan-trigger" class="org.springframework.scheduling.quartz.CronTriggerBean">
	<property name="jobDetail" ref="ftp-scan-jobDetail"></property>
	<property name="cronExpression" value="0 0/1 * * * ?"></property>
</bean>
<!-- 任务配置 -->
<bean id="ftp-scan-jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
	<property name="targetObject">
		<ref bean="ftp-scan-jobBean"/>
	</property>
	<property name="targetMethod">
		<value>execute</value>
	</property>
</bean>
```

### 类图

{% asset_img class_uml.png %}

### 看一些代码

```java
public class SchedulerFactoryBean extends SchedulerAccessor implements FactoryBean<Scheduler>, BeanNameAware, ApplicationContextAware, InitializingBean, DisposableBean, SmartLifecycle {
    private Class<?> schedulerFactoryClass = StdSchedulerFactory.class;
	
    public void afterPropertiesSet() throws Exception {
        ...

        SchedulerFactory schedulerFactory = (SchedulerFactory)BeanUtils.instantiateClass(this.schedulerFactoryClass);
		
		...
	}
}
```

默认使用的是 org.quartz.impl.StdSchedulerFactory。StdSchedulerFactory 代码就不贴了，从中可以看出，StdSchedulerFactory 从 quartz.properties 获取相关配置。

```ruby
org.quartz.scheduler.instanceName = DefaultQuartzScheduler
org.quartz.scheduler.rmi.export = false
org.quartz.scheduler.rmi.proxy = false
org.quartz.scheduler.wrapJobExecutionInUserTransaction = false

org.quartz.threadPool.class = org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount = 10
org.quartz.threadPool.threadPriority = 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread = true

org.quartz.jobStore.misfireThreshold = 60000

org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
```

{% note warning %}
再进一步看 StdSchedulerFactory 代码（这里就不贴了，太长了~），可以知道使用的线程池为 org.quartz.simpl.SimpleThreadPool，默认为 10 个线程，并且配置在同一个 SchedulerFactoryBean 下的定时器使用的是同一个线程池。
{% endnote %}

未完待续...
