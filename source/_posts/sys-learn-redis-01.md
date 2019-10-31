---
title: 【系统学习-redis】【一】概述和数据结构
categories:
  - redis学习
tags:
  - redis
date: 2019-10-31 23:39:57
---

## 概要

今天我们开始聊一下 redis，慢慢地聊一下。redis 我们经常用于缓存数据库、排行榜、计数器应用等，在平时开发使用中，已经变成不可或缺的应用了。我们先一起来聊一下 redis 的一些功能和提供的数据结构，所以我们今天聊的有以下几点：

- redis 杂谈
- 数据结构
- 一些全局的命令

<!-- more -->

## redis 杂谈

redis 是一种基于键值对(key-value)数据库，其中 value 可以为 string, hash, list, set, zset 等数据结构，还提供了键过期，发布订阅，事务，流水线等功能。redis 经常使用的场景有作为缓存数据库，排行榜，计数器应用，社交网络等

### redis 很快

redis 处理指令速度很快，也可以说是并发高，单机并发达到10W，实际需要真实测试，这个可以通过官方的压测工具 `redis-benchmark` 进行测试。那为什么 redis 很快呢，有以下几点原因

- redis 基于内存，所有数据都在内存中。内存的读写速度是很快的
- 单线程执行命令。避免了线程切换的消耗和线程之间资源的竞争
- 多路复用，nio(非阻塞io) 采用 epoll，不在 io 上浪费时间
- resp协议很简单。redis在tcp上设计了自己的resp协议，这个协议非常简单，解析请求和响应的时候就消耗少

### resp 协议

resp 协议规定了 redis 客户端和服务端之间的通信，协议的例子如下。这里有两条指令，一条是 `SELECT 0` 意思是使用第 0 个数据库，一条是 `set name sealde` 意思是设置一个值。可以看出协议中一条指令由三部分组成，第一部分`*2` 是命令由几个字符串组成，`$6` 指下一条字符串有 6 个字符，`SELECT` 指具体命令，即选择数据库，`$1` 指下一条字符串有 1 个字符，`0` 指具体的命令，即第 0 个。`set name sealde` 同样是这样的结构

```
*2
$6
SELECT
$1
0
*3
$3
set
$4
name
$6
sealde
```

resp 协议很简单，甚至没事干，可以自己写个客户端，在 TCP 连接上去发送和解析这样的字符串

## 数据结构

redis 数据结构一共有 string, list, set, zset, hash 这 5 种，可以通过[redis命令手册](https://www.redis.net.cn/order/)查看具体的命令。接下来我们来聊一下数据结构具体的命令，这里只聊比较常用的命令

### 字符串 string

字符串数据结构即 value 是字符串，命令我们大致分为下面几类

- 设置命令
  `set key value [EX seconds] [PX milliseconds] [NX|XX]` 这里的 value 可以是 字符串(JSON、XML等)、数字(整型和浮点数)、二进制(图片、音频和视频) 最大不能超过 512 M 
  `setnx key value` key不存在，设置成功返回1；失败返回0
- 获取命令
  `get key` 获取字符串
- 批量命令
  `mset key value [key value ...]` 批量设置 key-value
  `mget key [key ...]` 批量获取字符串
- 计数命令
  `incr key` 必须为整数，value自增1
  `decr key` 必须为整数，value自减1
  `incrby key increment` value 增加 increment 的数值
  `decrby key decrement` value 减去 decrement 的数值
  `incrbyfloat key increment` 浮点数增加 increment 的数值
- 字符串操作命令
  `append key value` 拼接字符串
  `strlen key` 计算字符串长度
  `getrange key start end` 截取字符串

下面看两个例子

`set age 23 ex 10`。这里 age 是 key，value 是 23，ex 指过期时间，这里 10 秒后过期
`setnx name test`。不存在 name 这个 key 时，返回 1 设置成功；否则返回失败 0

```
# 这个是字符串操作的例子
127.0.0.1:6379> set name hello
OK
127.0.0.1:6379> append name " world"
(integer) 11
127.0.0.1:6379> get name
"hello world"
127.0.0.1:6379> strlen name
(integer) 11
127.0.0.1:6379> getrange name 0 4
"hello"
```

### 哈希 hash

redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象。redis 中每个 hash 可以存储 2^32 - 1 键值对（40多亿）。常用的命令如下

- 设置、获取、删除
  `hset key field value` 设置值，一个 key 可以有多个 field，field 和 value 是隐射关系，就跟 java 的 map 一样
  `hget key field` 获取值
  `hdel key field [field ...]` 删除 field
- 计数
  `hlen key` 计算 key 下有多少对 field 和 value
  `hincrby key field increment` 对 field 映射的 value 增加 increment 的值
  `hincrbyfloat key field increment` 浮点数增加 increment 的值
- 批量操作
  `hmset key field value [field value ...]` 一次设置多个值
  `hmget key field [field ...]` 获取多个 field
  `hkeys key` 获取所有的 field
  `hvals key` 获取所有的 value
  `hgetall key` 获取所有的 field 和 value 的映射
- 判断 filed 是否存在
  `hexists key field`

我们保存一条数据库的记录，通常可以使用 hash 来保存，比如有一条 `id 123 name sealde age 19 city GuangZhou` 这样的记录可以这样保存

```
127.0.0.1:6379> hmset user:123 name sealde age 19 city GuangZhou
OK
127.0.0.1:6379> hgetall user:123
1) "name"
2) "sealde"
3) "age"
4) "19"
5) "city"
6) "GuangZhou"
127.0.0.1:6379> hget user:123 name
"sealde"
```

### 列表 list

存储多个有序的字符串，一个列表最多可以存 2^32-1 个元素，元素可以重复。redis 的 list 支持双端操作，即左右两边都可以放入元素和去除元素，其实跟 java 的 Deque 比较像。常用的命令如下

- 插入
  `rpush key value [value ...]` 右插元素，可以多个
  `lpush key value [value ...]` 左插元素，可以多个
  `linsert key BEFORE|AFTER pivot value` 在 pivot 之前或者之后插入 value
- 查找
  `lrange key start stop` 从左边范围查找
  `lindex key index` 根据下标查找元素
- 删除并返回(出队列)
  `lpop key` 队列左边弹出一个元素
  `rpop key` 队列右边弹出一个元素

我们来看一个例子，lpush 插入数据其实就是进入队列

```
127.0.0.1:6379> lpush ltest a b c d
(integer) 4
127.0.0.1:6379> lrange ltest 0 -1
1) "d"
2) "c"
3) "b"
4) "a"
127.0.0.1:6379> lindex ltest 0
"d"
127.0.0.1:6379> lindex ltest -1
"a"
127.0.0.1:6379> lpop ltest
"d"
```

### 集合 set

可以保存多个元素，元素不能重复，元素是无序的。常用的命令如下

- 添加、查询、删除
  `sadd key member [member ...]` 添加元素，一次可以添加多个元素
  `smembers key` 查询元素
  `srem key member [member ...]` 删除元素，一次可以删除多个元素
- 计数和判断是否存在
  `scard key` 统计元素个数
  `sismember key member` 判断元素是否存在集合中
- 集合操作
  `sinter key [key ...]` 求交集 `sinterstore destination key [key ...]` 求交集后存在 destination 中
  `sunion key [key ...]` 求并集 `sunionstore destination key [key ...]` 类似于上面
  `sdiff key [key ...]` 求差集 `sdiffstore destination key [key ...]` 类似于上面

集合可以做一些有爱好交集这样的需求，或者不能重复操作的需求。下面是一个添加和查询的例子

```
127.0.0.1:6379> sadd stest a b c a d
(integer) 4
127.0.0.1:6379> smembers stest
1) "d"
2) "c"
3) "b"
4) "a"
```

### 有序集合 zset

有序集合就是有序的集合 =。= 每个成员(member) 不能重复，每个成员对应一个分值(score)，常用于排行榜或点赞数等场景。我们先来对比一下 list、set、zset 使用上的区别

| 数据结构 | 是否允许元素重复 | 是否有序 | 有序的实现方式 | 应用场景 |
| :--: | :--: | :--: | :--: | :--: |
| list | 是 | 是 | 索引下标 | 时间轴，队列 |
| set | 否 | 否 | 无 | 标签，社交 |
| zset | 否 | 是 | 分值 | 排行榜，点赞数 |

有序集合常用的命令

- 插入、查询、删除
  `zadd key [NX|XX] [CH] [INCR] score member [score member ...]` 插入元素，nx 指不存在才能设置，xx 指存在时才能设置，ch 表示返回值为更新的数量，不加 ch 更新的个数是不会统计的，incr 自增1，score 分值，member 成员
  `zrem key member [member ...]` 删除成员
  `zrange key start stop [WITHSCORES]` 范围查询，withscores 指是否返回分值
  `zrevrange key start stop [WITHSCORES]` 在索引范围内 (start -- stop) 倒叙排
  `zrangebyscore key min max [WITHSCORES] [LIMIT offset count]` 在分值范围内 (min -- max) 排序返回
  `zrevrangebyscore key max min [WITHSCORES] [LIMIT offset count]` 在分值范围内 (max -- min) 倒叙排
  `zrank key member` 排名
  `zrevrank key member` 倒叙的排名
- 计数
  `zcard key` 统计成员数量

在排行榜场景中，zset 非常适合使用

## 一些全局的命令

- `keys pattern` 查找键，如 `keys user*`，特别的 `keys *` 列出所有的键
- `dbsize` 返回键的数量
- `exists key [key ...]` 查看键是否存在
- `del key [key ...]` 删除键
- `expire key seconds` 给键设置过期时间
- `ttl name` 查看键的过期时间，如果没有过期时间返回 -1
- `type key` 返回键的数据类型，如果键不存在返回 none

数据库管理命令

- `select index` 选择数据库，一共有16个数据库，0-15
- `flushdb` 删除当前 db 下的所有数据
- `flushall` 删除所有 db 下的数据

基本上我们已经聊完了常用的 redis 操作，我们结合业务灵活地使用这五种数据结构，比如一个排行榜功能，使用 zset 存储分值，点赞或者分享等给它加分值，每条数据一个用只能点赞一次可以使用 set 进行记录，查询自己数据的排名使用 zrevrank，跟前一名相比差多少分 `zrangebyscore key score +inf withscores limit 1 2`

今天就聊到这了，感觉没聊啥，当然 redis 还提供其他的功能，比如 发布订阅、HyperLogLog、bitmap、geo 地理位置、事务、lua脚本等，我们需要清楚这些都是提供什么样功能和适用的场景