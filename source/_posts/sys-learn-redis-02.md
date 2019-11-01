---
title: 【系统学习-redis】【二】工具和提供的功能
categories:
  - redis学习
tags:
  - redis
date: 2019-11-01 16:50:23
---

## 概要

最近晚霞特别好看，红成一片。今天我们聊一下 redis 提供的工具和功能，在我们使用 redis 过程中好有个什么时候可以借助工具帮住我们，所以今天我们聊一下

- redis-cli & redis-server
- 慢查询
- redis-benchmark 压测工具
- pipeline 管道
- 弱事务
- lua 原子操作
- 发布订阅
- 其他功能

<!-- more -->

## redis-cli & redis-server

redis-cli 是连接 redis 服务的客户端，redis-server 启动 redis 服务。这里我们聊一下常用的命令

- redis-cli
  `redis-cli -h 127.0.0.1 -p 6379` 连接 redis 数据库，-h 指 host 地址，-p 指端口，默认6379，另外 -a 是密码
  `redis-cli -r 10 -i 1 info |grep used_memory_human` 查看内存使用情况，每秒输出一次，一共输出 10 次。-r 指重复操作，-i 指在-r参数情况下间隔多长时间，info 查看 redis 信息
    ```
    ~ redis-cli -r 10 -i 1 info |grep used_memory_human
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    used_memory_human:821.46K
    ```
  `redis-cli keys "user:*" | xargs redis-cli del` 批量删除 user:* 匹配到的 key
- redis-server
  `redis-server ./redis.conf &` 指定配置文件启动，`&` 表示后台运行
  `redis-server --test-memory 1024` 检测系统能否提供 1G 内存给 redis，用于快速占满机器内存的测试。如果机器不能提供，会发现测试的速度变的很慢

更多的参数命令参考 `redis-cli --help` 和　`redis-server --help`

## 慢查询

跟 mysql 一样，当执行时间超过一定的阈值，会将耗时的命令记录起来。由于 redis 是单线程的，所以一条命令如果很耗时，会阻塞住后面的所有命令，我们来看一下怎么设置慢查询的阈值

- 动态设置，重启启动设置会丢失。直接在 redis-cli 连接上设置 `6379> config set slowlog-log-slower-than 10000`，这里设置阈值为 10ms
- 配置文件设置。修改配置文件 redis.conf
  ```
  # 设置为 10ms，特别地设置成 0 记录所有的记录，负数不记录
  slowlog-log-slower-than 10000

  # 设置慢查询记录的条数，慢查询记录也是队列，有一定的条数限制
  slowlog-max-len 128
  ```

**慢查询使用**

`slowlog len` 显示慢查询条数，`slowlog get` 获取慢查询命令

```
127.0.0.1:6379> slowlog len
(integer) 1
127.0.0.1:6379> slowlog get
1) 1) (integer) 0
   2) (integer) 1572578366
   3) (integer) 5
   4) 1) "get"
      2) "name"
   5) "127.0.0.1:55444"
   6) ""
```

一般生产上设置 slowlog-log-slower-than 为 10ms，如果并发要求高，可以设置 1ms。slowlog-max-len 可以设置上千条。慢查询日志最好写个脚本定期地存放起来，比如放到 mysql 里面

## redis-benchmark 压测工具

我们通常需要使用压测工具对我们的服务做基准测试，需要知道达到什么阈值下性能会出现问题，测试 redis 有一个官方提供的工具 redis-benchmark。我们先来看看几个例子

```
➜  ~ redis-benchmark -q
PING_INLINE: 143884.89 requests per second
PING_BULK: 143472.02 requests per second
SET: 141643.06 requests per second
GET: 145348.83 requests per second
INCR: 146627.56 requests per second
LPUSH: 146198.83 requests per second
RPUSH: 146412.88 requests per second
LPOP: 146198.83 requests per second
RPOP: 146412.88 requests per second
SADD: 146627.56 requests per second
HSET: 147058.83 requests per second
SPOP: 146198.83 requests per second
LPUSH (needed to benchmark LRANGE): 147058.83 requests per second
LRANGE_100 (first 100 elements): 89206.06 requests per second
LRANGE_300 (first 300 elements): 35842.29 requests per second
LRANGE_500 (first 450 elements): 24727.99 requests per second
LRANGE_600 (first 600 elements): 18491.12 requests per second
MSET (10 keys): 112107.62 requests per second
```

这里是我本机测试，注意生成压测的时候需要加上 host，port 和 pwd，`redis-benchmark -q` -q 是 quiet 的意思，只输出每秒查询的次数。还可以指定并发数进行测试

- `redis-benchmark -q -c 1000 -n 100000` 1000 个并发连接，每个连接 10w次请求
- `redis-benchmark -q -d 1000` 测试 1000字节 大小数据包的 set/get 情况
- `redis-benchmark -q -t set,lpush --csv` 只测试 set,lpush 的性能，输出为 csv 格式
- `redis-benchmark -r 10000 -n 10000 eval 'return redis.call("ping")'` 执行脚本语句

## pipeline 管道

redis 客户端(比如 Jedis)执行一条命令，过程是 发送命令到服务端 -》命令进入队列 -》 服务端执行命令 -》 返回执行结果给客户端，客户端在每执行命令的时候都需要往返的网络开销。pipeline 的作用是一次可以执行多条命令，而不用多次地发送请求，pipeline 的出现就是为了节省网络请求的开销，网络开销的时间可能比服务端执行命令的时间还要长，特别客户端和服务端网络延迟大的情况更明显

```java
public static void main( String[] args ) {
    Jedis jedis = new Jedis("127.0.0.1", 6379);
    Pipeline pipeline = jedis.pipelined();
    pipeline.set("ptest1", "t");
    pipeline.sadd("pztest1", "p1", "p2", "p3");
    Response<Long> res = pipeline.dbSize();

    pipeline.sync();
    System.out.println(res.get());
}
```

在客户端使用 pipeline 比如 Jedis，将一组命令组装起来，发送给 redis 服务端，完成批量操作，其实发送的就是根据 resp 协议组装出来的文本。其实原生命令也有批量操作的，比如 `mset`、`hmset` 等，他们的区别

- 原生批命令是原子的，pipeline 不具有原子性
- 原生批命令只能单一的操作，pipeline 可以有多条不同的命令
- 原生批命令是服务端实现的，pipeline 是需要服务端和客户端共同完成的

一般我们使用 pipeline 需要主要组装的命令不能太多，因为 redis 单线程的关系，有阻塞其他命令的风险。如果有大量的命令需要执行，可以拆分成多个小的 pipeline 去执行。pipeline 还可以结合 redis 的事务执行命令，即将命令放在 `multi` 和 `exec` 之间执行，`multi` 指事务开启，`exec` 指事务结束

## 弱事务

redis 提供的事务是弱事务性的，我们看一个例子，这里第二条语句出错了，但是第一条和第三条语句还是执行成功了，这个说明事务没有回滚，这就是弱事务性
什么样的情况会回滚呢，比如这里第二条语句是 `ssdfa` 这样的语法错误，那么事务直接失败，语句不执行了，或者 `discard` 显示停止提交事务

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set multi:test ooo
QUEUED
127.0.0.1:6379> zincrby multi:test q notexists
QUEUED
127.0.0.1:6379> set multi:not ppp
QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) ERR value is not a valid float
3) OK
127.0.0.1:6379> get multi:test
"ooo"
```

## lua 原子操作

lua 是一门小巧的脚本语言，用 C 语言写的，速度很快。redis 可以将 lua 脚本加载到内存中，然后调用 lua 脚本。使用 lua 脚本有以下几个好处：

- 减少网络开销
- 原子操作
- 复用性

这里不对 lua 语言的语法进行讨论，我们先来看一个简单的例子。这里通过 `redis.call` 调用 redis 的命令，通过 `KEYS[1]` 获取到参数，1 表示第一个参数

```
127.0.0.1:6379> set name sealde
OK
// eval + 脚本 + 键个数 + 键
127.0.0.1:6379> eval "return redis.call('get',KEYS[1])" 1 name
"sealde"
```

### redis lua脚本管理

- `script load "具体的脚本"` 将脚本加载到 redis 中，会返回 sha 签名
- `evalsha sha1 numkeys key [key ...] arg [arg ...]` 执行 sha1 脚本，numkeys 指 key 的数量
- `script exists sha1` 判断脚本是否存在
- `script kill` 停止正在运行的脚本
- `script flush` 清空 redis 存储的脚本

我们接下来看一个例子，这里我们写好一个 lua 脚本 `accesslimit.lua`，它的功能是限制同一个 ip 一定时间内访问次数，其中 KEYS[1] 指 ip 地址，KEYS[2] 指限制访问的次数，KEYS[3] 指过期时间。可以看到，`script load` 将脚本加载到 redis 并返回了 sha1，然后通过 `evalsha sha1 numkeys key ...` 执行该脚本
效果是，刚开始 2 次访问返回 1，接下来访问返回 0，查过 5 秒后又可以继续访问，是不是挺简单的呀

```
➜  lua cat accesslimit.lua 
-- KEYS[1] ip
local key = "accesslimit:"..KEYS[1]

if (redis.call("exists", key) == 1)
then
    local times = redis.call("incr", key)
    -- KEYS[2] limit times
    if (times > tonumber(KEYS[2])) then
 	return 0
    end
else
    -- KEYS[3] expire time
    redis.call("set", key, 1, "EX", tonumber(KEYS[3]))
end
return 1
➜  lua redis-cli script load "$(cat $(pwd)/accesslimit.lua)"
"ae251809e1e2f5c68bbf6b1196e3568920d5c34c"
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 1
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 1
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 0
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 0
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 0
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 0
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 0
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 0
➜  lua redis-cli evalsha ae251809e1e2f5c68bbf6b1196e3568920d5c34c 3 127.0.0.1 2 5
(integer) 1
```

## 发布订阅

发布订阅即发布一个消息到某个 channel，订阅了该 channel 就可以收到该消息。我们先来看一个例子，这里我们启动了两个 redis-cli 去连接

订阅者

```
127.0.0.1:6379> subscribe channel:test
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "channel:test"
3) (integer) 1

-- 当 channel:test 有消息的时候
1) "message"
2) "channel:test"
3) "hello world!"
```

发布者

```
127.0.0.1:6379> publish channel:test "hello world!"
(integer) 1
```

最主要的命令就是 `publish` 和 `subscribe`，另外还有不同模式的订阅，退订等

## 其他功能

redis 还提供其他功能，虽然今天不聊怎么具体使用或者原理，但是大概聊一下有什么作用，以后有相关的需求可以使用上来

- 地理位置(geo)，将经纬度坐标数据存到 redis 中，可以查询坐标之间的距离，查询指定范围内的有序坐标。可以做附近的人，物流等功能
- BitMap 通过二进制位存储状态信息，节省内存。根据偏移量进行设置状态，统计等。可以做活跃用户统计，用户行为统计等功能
- HyperLogLog 通过概率算法统计数据，节省内存，可以统计很大数量级的数据。可以用做访问量，活跃用户等

好啦，今天就聊到这了，还有很多可以慢慢探索的地方，这里只是浅显的聊一下，下次再聊点其他的