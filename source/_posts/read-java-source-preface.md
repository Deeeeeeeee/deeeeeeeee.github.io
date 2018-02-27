---
title: 阅读源码系列--双重校验锁
categories:
  - null
tags:
  - null
date: 2018-02-27 22:46:00
---


双重校验锁：即在代码 synchronized 代码块之前和代码块开始，都对一个变量进行校验

javax.xml.parsers.FactoryFinder 类里面有个方法 static <T> T find(Class<T> type, String fallbackClassName)，其中有一段：

```java
try {
    if (firstTime) {
        synchronized (cacheProps) {
            if (firstTime) {
            ...
```

<!-- more -->

其中 firstTime 和 cachePros 为。firstTime 判断是否已经缓存过，cachePros 缓存 properties
```java
/**
 * Flag indicating if properties from java.home/lib/jaxp.properties
 * have been cached.
 */
static volatile boolean firstTime = true;

/**
 * Cache for properties in java.home/lib/jaxp.properties
 */
private static final Properties cacheProps = new Properties();
```

{% note warning %}
那么，问题来了，为什么要写成这么蛋疼的加锁形式呢？
{% endnote %}

## 不加锁的情况
```java
try {
	if (firstTime) {
	...
```

当有多个线程调用这一段时，有种情况会出现问题，cachePros 已经缓存了，但是另外一个线程执行到了 if(firstTime)，此时 firstTime 还是为 true。那么就出现问题了

## 加锁的情况
```java
try {
	synchronized (cacheProps) {
		if (firstTime) {
```

这种情况效率会比较低一点，因为同一时间只允许一个线程执行这段代码。但是可以在前面添加一个判断，让程序只在第一次调用时阻塞

```java
try {
    if (firstTime) {
        synchronized (cacheProps) {
            if (firstTime) {
            ...
```

## 使用 volatile 同步变量并防止指令重排
jvm 优化指令时，有可能会重排指令，虽然感觉这里不需要防止指令重排。啊哈哈哈