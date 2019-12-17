---
title: 【系统学习-redis】【三】高可用和集群
categories:
  - redis学习
tags:
  - redis
date: 2019-11-06 17:55:15
---

## 概要

时间过的很快，今天我们来聊一下 redis 中的高可用和集群，如果我们是自己搭建服务，不局限于 redis，很多时候都要考虑服务的高可用，当服务出现性能问题，很多时候我们都要考虑做集群。那么怎么做可以实现 redis 的高可用呢，今天我们来聊一下以下几点：

- redis 持久化
- redis 主从复制
- redis 哨兵模式
- redis cluster

## redis 持久化

我们先来聊一下 redis 的持久化，因为 redis 是基于内存的，如果没有持久化，那么重启 redis，所有数据都会丢失，并且 redis 主从复制也跟持久化有关系。那么持久化是什么，持久化就是将内存中的数据同步到磁盘来避免数据丢失

### 持久化方式

redis 持久化方式有两种，一种是 rdb 持久化，也叫快照，另外一种是 aof 持久化。rdb 持久化是全量复制，aof 持久化是增量复制

```
sealde-MS-7B23# tree
.
├── appendonly.aof
└── dump.rdb
```

### rdb 持久化方式

rdb 持久化把数据生成 .rdb 文件，比如 `/var/lib/redis/dump.rdb`。rdb 持久化有手动触发和自动触发，自动触发我们就不聊了，来看一下手动触发，手动触发有 `save` 和 `bgsave` 两个命令

- save 命令，会阻塞 redis，直到 rdb 持久化过程完成，可能会造成查时间的阻塞，线上不建议使用
- bgsave 命令，进程执行 fork 操作创建子线程，由子线程完成持久化，阻塞时间很短

```
127.0.0.1:6379> save
OK
127.0.0.1:6379> bgsave
Background saving started
```

因为 rdb 持久化是压缩后的二进制文件，而且是全量的，所以适用于备份、灾难恢复等。那么我们怎么备份和恢复呢，其实只要 `bgsave` 持久化到 dump.rdb，恢复的时候放到 redis.conf 中 `dir` 配置的路径下，然后重启，另外 `config set dir /xxx/xxx` 可以设置 rdb 文件保存的路径。**这里需要注意的是如果同时开启 rdb 和 aof 持久化，恢复的时候需要特别注意，接下来会聊到这个**

### aof 持久化方式

rdb 持久化不是实时的，而且是全量的，每次操作都会创建新线程，频繁操作成本较高。redis 提供的 aof 持久化来解决这个问题，开启持久化可以执行命令 `config set appendonly yes` 或者修改配置文件 `/etc/redis/redis.conf`，设置 `appendonly yes`。aof 持久化默认是不开启的

#### aof 持久化流程

- 所有写入命令会 append 追加到 aof_buf 缓冲区中
- aof_buf 缓冲区向硬盘做 sync 同步
- 随着AOF文件越来越大，需定期对 aof 文件 rewrite 重写，达到压缩
- 当 redis 服务重启，可 load 加载 aof 文件进行恢复

#### aof 持久化相关配置

- appendonly yes 开启持久化
- appendfsync always/everysec/no 推荐使用 everysec，即每秒持久化一次
- no-appendfsync-on-rewrite no 正在 rdb 持久化，要不要停止 aof 持久化。考虑到是否 rdb 和 aof 同时进行会对磁盘有大量的 I/O 操作，aof 的 fsync 可能会阻塞很长时间，所以默认是 no 的，但有需要可以改成 yes
- auto-aof-rewrite-percentage 100 // aof 文件增长率为 100% 时，rewrite
- auto-aof-rewrite-min-size 64mb // aof 文件至少超过 64mb 时才开始 rewrite
- aof-load-truncated yes 会不会加载可能被截断的 aof 文件，yes 表示会，比如 aof 正在持久化，因为一些问题，导致没有有些字节没有写到磁盘，yes 的时候会丢弃有问题的命令然后加载成功，no 会直接报错

### rdb & aof

由于 rdb 和 aof 的特性不一样，各有优势，所以一般是一起使用，但 redis 加载数据机制有个坑的地方，就是如果开启了 aof 或者同时开启了 rdb 和 aof，redis 都会加载 aof 文件，如果没有 aof 文件则会创建一个空的 aof 文件。**如果刚开始使用的是 rdb方式，然后修改配置文件 appendonly yes，接着直接重启，这样 redis 会创建一个空的 aof 文件，会直接让你的数据丢失，所以千万不能这么干！！！**
我们可以通过动态配置来规避这个问题，即执行命令 `config set appendonly yes`，然后修改配置文件，接下来就可以随便重启了，因为动态配置之后，redis 会将 rdb 数据加载到 aof 文件中。所以我们做灾备恢复的思路同样如此，先在配置文件 appendonly no，然后拷贝 dump.rdb，接下来启动，动态配置 config set appendonly yes，再接下修改配置文件 appendonly yes，这样就完成了

## redis 主从复制

redis 配置主从的方式有两种

- 通过修改配置(这里包括修改配置文件和动态配置)，在配置中添加 `slaveof xxxx.xxx.xx.xxx 6379`，xxxx.xxx.xx.xxx 为 ip 地址
- 启动的时候添加参数 `redis-server --slaveof xxxx.xxx.xx.xxx 6379`

动态建立主从和断开主从，执行命令 `6381:> slaveof 192.169.0.1 6380` 和 `6381:> slaveof no one`。这里有一个[redis主从测试的例子](https://github.com/Deeeeeeeee/ops_demos/tree/master/redis/master-slave)

```
├── docker-compose.yml
├── redis6380.conf
└── redis6381.conf
```

redis6380 和 redis6381 配置几乎一样，只是 redis6381.conf 加了 `slaveof 192.169.0.1 6380`，作为 6380 的从节点，当 6380 数据改变时，会同步到 6381 从节点

```shell
# 6380 设置值 name
127.0.0.1:6380> dbsize
(integer) 0
127.0.0.1:6380> set name test
OK

# 6381 当 6380 设置 name 值后，数据会同步过来
127.0.0.1:6381> dbsize
(integer) 0
127.0.0.1:6381> get name
"test"
```

另外，还有一个还有一个参数 `repl-disable-tcp-delay yes`，默认是 no，当为 no 时 master 数据更新会立即同步到 slave 中；当为 yes 时 40 ms 才会发送一次。所以跨机房或者网络环境较差的情况下，建议为 `yes`
最后我们再来聊聊同步的过程，当 slave 节点启动时

- 保存 master 节点信息
- 主从建立 socket 连接
- 发送 ping 命令
- 权限验证
- 同步数据
- 持续同步数据

## redis 哨兵模式

哨兵模式是 redis 迈向高可用的重要一步，高可用即当一个服务挂了，可以自动切换到另一个服务，提供不间断的服务。我们上面聊的主从复制，其实有一个很大的问题，就是如果 master 节点挂了，需要手工进行恢复，这样即麻烦又有服务较长时间的间断，哨兵模式就是为了解决这个问题

### 哨兵模式机制

{% asset_img redis-sentinel.png %}

[redis sentinel 官方文档](https://redis.io/topics/sentinel) 哨兵模式的机制：哨兵不停地监控着 redis 实例，当一个哨兵发现 master 实例下线，会通知其他哨兵进行确认，当多个哨兵(达到 quorum即法定人数)监控到实例下线，则认为该实例下线，然后哨兵之间会投票出领导者，然后由领导者进行故障处理。如果刚好下线的实例是 master 节点，那么会让其他实例成为 master，这个过程等会再聊
那么**哨兵监控的内容**有哪一些:

- 每隔 10s 会对所有 redis 实例发一次 info 请求，主要是连接情况，内存 CPU 使用情况
- 每隔 5s 利用 master 的 pulish/subscrib，发布自己的信息，比如 ip、port、runid、处理故障的能力等
- 每隔 1s 对所有 redis 实例和其他哨兵发一次 ping 请求

哨兵监控实例，还有一个主观下线和客观下线的概念。这里有两个值需要注意，一个是 `is-master-down-after-milliseconds` 判断是否下线的时间，如果设置成30s，就算 ping 在前 28s 内没有响应，最后 1s 有响应都视为实例可以正常工作；另一个是 `is-master-down-by-addr <ip> <port>` 用来在哨兵直接确认 master 节点是否下线

- 主观下线即哨兵发现 redis 实例下线了，但没有达到配置的 quorum
- 客观下线是针对 master 节点的，当达到了 quorum 的哨兵监控到 master 节点下线了，会认为客观下线了。所以 slave 节点和其他哨兵没有客观下线这个概念

### 故障转移

哨兵模式的故障转移会有以下几个步骤

- 确认 master 客观下线
- 选举 leader(领导者) ，leader 进行故障转移。其他哨兵称为 observers(观察着)
- leader 选择一个 slave 节点成为 master 节点
- 晋升的 slave 节点执行 `slaveof no one`
- observers 看到晋升的 slave 节点成为 master 节点，明白故障转移开始了
- 其他 slave 节点通过 `slaveof <host> <port>` 改变他们的 master
- 当其他 slave 重新配置之后，leader 终止故障转移，将监控表(table of monitored)中移除旧 master 并用新 master 替代
- observers 检测到所有 slave  重新配置后，同样从 table 中移除 old master 并用 new master 代替

leader 的选举原理跟判断 master 节点客观下线是一样的，其中不能进行故障转移的哨兵，判断为主观下线、连接不上、ping 超过一定阈值并最迟响应的哨兵不能参与选举。leader 需要 ping 响应时间 5s 以内，runid 最小，并且在 Pub/Sub 的 channel 中表示有能力处理故障转移，最后不是 DISCONNECTED 状态。更多的条件判断请看 [redis sentinel 官方文档](https://redis.io/topics/sentinel)
**另外 `replica-priority` 可以设置 slave 节点成为 master 的优先级**

### 配置以及测试

[官网 sentinel.conf](http://download.redis.io/redis-stable/sentinel.conf)，官方的配置文件可以下载下来参考一下。在这里有个 [redis-sentinel 三节点配置](https://github.com/Deeeeeeeee/ops_demos/tree/master/redis/sentinel/three-boxes)，模型如下，即模拟三台机器，每台机器有一个 redis 实例和 redis sentinel

```
       +----+
       | M1 |
       | S1 |
       +----+
          |
+----+    |    +----+
| R2 |----+----| R3 |
| S2 |         | S3 |
+----+         +----+

Configuration: quorum = 2
```

目录结构，这里有三个配置文件夹，代表三台机器的配置，每台配置通过 `docker-compose up -d` 启动，这里只是模拟，生产环境配置参考上面的 `官方 sentiel.conf` 和 `redis sentinel 官方文档`

```
├── box1
│   ├── docker-compose.yml
│   ├── redis6380.conf
│   └── sentinel26380.conf
├── box2
│   ├── docker-compose.yml
│   ├── redis6381.conf
│   └── sentinel26381.conf
└── box3
    ├── docker-compose.yml
    ├── redis6382.conf
    └── sentinel26382.conf
```

sentinel 的配置文件如下，需要指明监控的 master 节点，如果有密码需要添加密码

```conf
port 26380
daemonize no
pidfile "/var/run/redis-sentinel.pid"
logfile ""
# 监控的 master 节点和密码
sentinel monitor mymaster 192.169.0.1 6380 2
sentinel auth-pass mymaster 123456
# 使用 docker 或者 nat 时，需要配置
sentinel announce-ip "192.169.0.1"
sentinel announce-port 26380
```

另外 docker-compose.yml 的配置如下，这里启动了两个 docker 容器，分别启动 redis 实例和 redis-sentinel

```yml
version: '3.0'
services:
  redis-6380-boxes1:
    image: redis
    container_name: redis-6380-boxes1
    ports:
      - "6380:6380"
    volumes:
      - ./redis6380.conf:/etc/redis/redis.conf
    privileged: true
    command: redis-server /etc/redis/redis.conf
    
  sentinel-6380-boxes1:
    image: redis
    container_name: sentinel-6380-boxes1
    ports:
      - "26380:26380"
    volumes:
      - ./sentinel26380.conf:/etc/redis/sentinel.conf
    privileged: true
    command: redis-server /etc/redis/sentinel.conf --sentinel
```

sentinel 配置效果如下

```shell
# redis sentinel 启动完后，生成自己的 ID，并开始监控
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:29:03.025 * Running mode=sentinel, port=26380.
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:29:03.025 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:29:03.028 # Sentinel ID is 631dbd0d112dc52dc10b03f037d1ed430c22f4c4
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:29:03.028 # +monitor master mymaster 192.169.0.1 6380 quorum 2

# 当有其他 slave 节点加入，会同步数据给 slave 节点。slave 和 replica 是一样的
redis-6380-boxes1       | 1:M 17 Nov 2019 16:29:12.606 * Replica 192.168.112.1:6381 asks for synchronization
redis-6380-boxes1       | 1:M 17 Nov 2019 16:29:12.606 * Full resync requested by replica 192.168.112.1:6381
redis-6380-boxes1       | 1:M 17 Nov 2019 16:29:12.606 * Starting BGSAVE for SYNC with target: disk
redis-6380-boxes1       | 1:M 17 Nov 2019 16:29:12.606 * Background saving started by pid 18
redis-6380-boxes1       | 18:C 17 Nov 2019 16:29:12.610 * DB saved on disk
redis-6380-boxes1       | 18:C 17 Nov 2019 16:29:12.611 * RDB: 0 MB of memory used by copy-on-write
redis-6380-boxes1       | 1:M 17 Nov 2019 16:29:12.702 * Background saving terminated with success
redis-6380-boxes1       | 1:M 17 Nov 2019 16:29:12.703 * Synchronization with replica 192.168.112.1:6381 succeeded

# 当有其他 sentinel 节点加入，会记录 sentinel 信息到 sentinel.conf 配置文件
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:29:13.076 * +slave slave 192.168.112.1:6381 192.168.112.1 6381 @ mymaster 192.169.0.1 6380
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:29:14.675 * +sentinel sentinel 6d06e814eb696333ba86771ee0d7d69ea10eec46 192.169.0.1 26381 @ mymaster 192.169.0.1 6380

# 当 master 节点挂了的时候，sentinel 先会判断主动下线，然后判断客观下线，投票 leader，故障转移，重新调整 master
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.644 # +sdown master mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.706 # +odown master mymaster 192.169.0.1 6380 #quorum 3/2
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.706 # +new-epoch 1
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.706 # +try-failover master mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.717 # +vote-for-leader 6d06e814eb696333ba86771ee0d7d69ea10eec46 1
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.724 # 631dbd0d112dc52dc10b03f037d1ed430c22f4c4 voted for 6d06e814eb696333ba86771ee0d7d69ea10eec46 1
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.724 # d26ff112e53779f599f317d424cd8d01ad3feae9 voted for 6d06e814eb696333ba86771ee0d7d69ea10eec46 1
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.817 # +elected-leader master mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.817 # +failover-state-select-slave master mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.889 # +selected-slave slave 192.168.112.1:6382 192.168.112.1 6382 @ mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.889 * +failover-state-send-slaveof-noone slave 192.168.112.1:6382 192.168.112.1 6382 @ mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:19.980 * +failover-state-wait-promotion slave 192.168.112.1:6382 192.168.112.1 6382 @ mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:20.173 # +promoted-slave slave 192.168.112.1:6382 192.168.112.1 6382 @ mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:20.173 # +failover-state-reconf-slaves master mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:20.228 * +slave-reconf-sent slave 192.168.112.1:6381 192.168.112.1 6381 @ mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:20.901 # -odown master mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:21.204 * +slave-reconf-inprog slave 192.168.112.1:6381 192.168.112.1 6381 @ mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:21.204 * +slave-reconf-done slave 192.168.112.1:6381 192.168.112.1 6381 @ mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:21.274 # +failover-end master mymaster 192.169.0.1 6380
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:21.274 # +switch-master mymaster 192.169.0.1 6380 192.168.112.1 6382
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:21.274 * +slave slave 192.168.112.1:6381 192.168.112.1 6381 @ mymaster 192.168.112.1 6382
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:21.275 * +slave slave 192.169.0.1:6380 192.169.0.1 6380 @ mymaster 192.168.112.1 6382
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:21.915 * +slave slave 192.168.144.1:6381 192.168.144.1 6381 @ mymaster 192.168.112.1 6382
sentinel-6381-boxes2    | 1:X 17 Nov 2019 16:49:51.281 # +sdown slave 192.169.0.1:6380 192.169.0.1 6380 @ mymaster 192.168.112.1 6382

# 重启原来的 mater 节点，已经变成 slave 节点了。这个需要自己手动重启
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:54:44.079 # -sdown slave 192.169.0.1:6380 192.169.0.1 6380 @ mymaster 192.168.112.1 6382
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:54:54.023 * +convert-to-slave slave 192.169.0.1:6380 192.169.0.1 6380 @ mymaster 192.168.112.1 6382
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:54.024 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:54.024 * REPLICAOF 192.168.112.1:6382 enabled (user request from 'id=4 addr=192.168.112.1:37484 fd=10 name=sentinel-631dbd0d-cmd age=10 idle=0 flags=x db=0 sub=0 psub=0 multi=3 qbuf=153 qbuf-free=32615 obl=36 oll=0 omem=0 events=r cmd=exec')
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:54.026 # CONFIG REWRITE executed with success.
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.000 * Connecting to MASTER 192.168.112.1:6382
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.000 * MASTER <-> REPLICA sync started
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.000 * Non blocking connect for SYNC fired the event.
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.001 * Master replied to PING, replication can continue...
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.002 * Trying a partial resynchronization (request 2421560b4156a1f210937207d4a424888bbb6427:1).
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.004 * Full resync from master: 99e9776ff1f78c77e259087bf3fbf3a3628a1ae7:306747
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.004 * Discarding previously cached master state.
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.065 * MASTER <-> REPLICA sync: receiving 178 bytes from master
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.065 * MASTER <-> REPLICA sync: Flushing old data
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.066 * MASTER <-> REPLICA sync: Loading DB in memory
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.066 * MASTER <-> REPLICA sync: Finished with success
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.067 * Background append only file rewriting started by pid 18
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.093 * AOF rewrite child asks to stop sending diffs.
redis-6380-boxes1       | 18:C 17 Nov 2019 16:54:55.093 * Parent agreed to stop sending diffs. Finalizing AOF...
redis-6380-boxes1       | 18:C 17 Nov 2019 16:54:55.093 * Concatenating 0.00 MB of AOF diff received from parent.
redis-6380-boxes1       | 18:C 17 Nov 2019 16:54:55.094 * SYNC append only file rewrite performed
redis-6380-boxes1       | 18:C 17 Nov 2019 16:54:55.095 * AOF rewrite: 0 MB of memory used by copy-on-write
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.101 * Background AOF rewrite terminated with success
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.101 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
redis-6380-boxes1       | 1:S 17 Nov 2019 16:54:55.102 * Background AOF rewrite finished successfully
sentinel-6380-boxes1    | 1:X 17 Nov 2019 16:55:02.210 * +slave slave 192.168.144.1:6380 192.168.144.1 6380 @ mymaster 192.168.112.1 6382
```

配置的时候需要注意的是

- 当有配置密码的时候，master 节点也需要配置密码 `masterauth <pwd>`，因为如果故障恢复后，再启动这个旧的 master 节点需要密码
- 如果使用 docker 或者 nat 技术的话，需要配置 `sentinel announce-ip "ip地址"` 和 `sentinel announce-port 端口`

另外需要注意的是，高可用是会有数据丢失的，因为 master 挂了之后，需要一定的时间进行故障转移，这段时间写入到 old master 的数据将会丢弃。当然可以配置确认故障的时候，禁止写入数据到 old master，但也会有一段时间 redis 服务不可用的问题

## redis cluster