---
title: 【系统学习-mysql】【一】架构和存储引擎
categories:
  - mysql 学习
tags:
  - mysql
date: 2019-10-12 09:21:06
---

## 概要

前段时间一直在聊并发编程，估计都腻了吧，其实还有一些并发编程的东西可以聊一下。我们还是先换个主题吧，后面有机会再聊一下并发编程。我们今天来聊一下 Mysql，先来聊一下抽象一点的东西，聊聊 mysql 的大概架构和存储引擎。所以今天聊的内容有

- 性能衡量标准
- 逻辑架构
- 存储引擎

<!-- more -->

## 性能衡量标准

tps 和 qps

- TPS(Transactions Per Second）每秒传输的事物处理个数：TPS = (COM_COMMIT + COM_ROLLBACK)/UPTIME
- QPS(Questions Per Second) 每秒查询速度：QPS = QUESTIONS/UPTIME

我们国内比较流行这样的查询去衡量数据库的使用情况，也算一种监控吧，当然还有很多指标可以监控

### 计算 tps

由于 mysql5.7 之后我没找到 com_commit 和 com_rollback 这两个系统状态存在哪张表里，所以就将就一下吧。当然也可以设置 show_compatibility_56 解决，只是我懒～毕竟我没有要成为 DBA

```sql
use performance_schema;

show global status like 'com_commit';
show global status like 'com_rollback';
select VARIABLE_VALUE into @uptime from global_status where VARIABLE_NAME = 'uptime';
-- 这里com_commit 和 com_rollback 换上面查出来的结果
select (com_commit + com_rollback)/@uptime;
```

### 计算 qps

```sql
use performance_schema;

select VARIABLE_VALUE into @questions from global_status where VARIABLE_NAME = 'Questions';
select VARIABLE_VALUE into @uptime from global_status where VARIABLE_NAME = 'uptime';
select @questions/@uptime;
```
### mysqlslap 压测工具

mysqlslap 这个压测工具是自带的工具，一般用来给 mysql 服务器做基准测试，协助后续使用中的监控和优化。`mysqlslap --help` 或 `man mysqlslap` 可以查看使用手册，不过这个是在 unix 类系统上的命令，其实 [mysql 文档](https://dev.mysql.com/doc/refman/5.7/en/mysqlslap.html) 上也可以看。当然我们可以直接看看是怎么使用的

```shell
mysqlslap -uroot -p123456 --concurrency=1,50,100,200 --iterations=3 --number-char-cols=5 --number-int-cols=5 --auto-generate-sql --auto-generate-sql-add-autoincrement --engine=myisam,innodb --create-schema='test'

// 输出结果
Benchmark
	Running for engine myisam
	Average number of seconds to run all queries: 0.000 seconds
	Minimum number of seconds to run all queries: 0.000 seconds
	Maximum number of seconds to run all queries: 0.000 seconds
	Number of clients running queries: 1
	Average number of queries per client: 0

Benchmark
	Running for engine myisam
	Average number of seconds to run all queries: 0.023 seconds
	Minimum number of seconds to run all queries: 0.010 seconds
	Maximum number of seconds to run all queries: 0.049 seconds
	Number of clients running queries: 50
	Average number of queries per client: 0

Benchmark
	Running for engine myisam
	Average number of seconds to run all queries: 0.025 seconds
	Minimum number of seconds to run all queries: 0.018 seconds
	Maximum number of seconds to run all queries: 0.036 seconds
	Number of clients running queries: 100
	Average number of queries per client: 0

Benchmark
	Running for engine myisam
	Average number of seconds to run all queries: 0.111 seconds
	Minimum number of seconds to run all queries: 0.047 seconds
	Maximum number of seconds to run all queries: 0.161 seconds
	Number of clients running queries: 200
	Average number of queries per client: 0

Benchmark
	Running for engine innodb
	Average number of seconds to run all queries: 0.029 seconds
	Minimum number of seconds to run all queries: 0.029 seconds
	Maximum number of seconds to run all queries: 0.029 seconds
	Number of clients running queries: 1
	Average number of queries per client: 0

Benchmark
	Running for engine innodb
	Average number of seconds to run all queries: 0.159 seconds
	Minimum number of seconds to run all queries: 0.153 seconds
	Maximum number of seconds to run all queries: 0.172 seconds
	Number of clients running queries: 50
	Average number of queries per client: 0

Benchmark
	Running for engine innodb
	Average number of seconds to run all queries: 0.342 seconds
	Minimum number of seconds to run all queries: 0.310 seconds
	Maximum number of seconds to run all queries: 0.392 seconds
	Number of clients running queries: 100
	Average number of queries per client: 0

Benchmark
	Running for engine innodb
	Average number of seconds to run all queries: 0.432 seconds
	Minimum number of seconds to run all queries: 0.382 seconds
	Maximum number of seconds to run all queries: 0.467 seconds
	Number of clients running queries: 200
	Average number of queries per client: 0
```

可以看到 myisam 引擎还是比 innodb 的查询速度快不少的，那稍微聊一下 mysqlslap 一些参数的意义吧，当然也可以看上面那份 mysql文档，有更详细的说明

```
-u 用户 -p 密码
--concurrency 查询并发数  --iterations 迭代次数
--number-char-cols 多少列字符类型的数据
--number-int-cols  多少列整型的数据
--auto-generate-sql 自动生成sql语句
--auto-generate-sql-add-autoincrement 生成的表添加自增列
--engine 需要测试的引擎
--create-schema schema名称(数据库名称)
```

当然，还有一些需要开启 mysql debug 才能测试的，比如 cpu 和 内存信息等，只是再一次，因为我毕竟不想成为 DBA，我又懒了，大家可以去网上查找一下怎么开启

## 逻辑架构

{% asset_img mysql-engine-overview.png %}

mysql 的逻辑架构可以分为以下四个层次

- 连接层
  - 图中上面 Connectors， Connection Pool 连接池 和左边 管理和安全等组成了连接层
  - mysql 是多用户连接的，每连接一个就会创建一个线程；接着会进行安全验证等
- 服务层
  - 图中四个框框，SQL Interface 接口，Parser 解析器，Optimizer 优化器，Caches & Buffers 缓存 这几个部分组成
  - sql 执行(查询)的过程。先判断是否命中缓存
    - 如果没有命中交给 Parser 进行语义分析，然后再给 Optimizer 优化，接着执行语句返回结果
    - 如果命中缓存，则直接返回结果，或者直接拿到解析优化后的执行语句
- 存储引擎层
  - 图中中下长长的框 Pluggable Storage Engines 是存储引擎层
- 文件层

### mysql 缓存

mysql 缓存分为

- sql 语句缓存。节省执行计划的时间
- 缓存数据(生产不建议开启)
  - 不建议开启是因为，很多操作都会引起缓存失效，这会给数据库带来更多的压力，一般建议在应用上做缓存，比如使用 redis 等
  - 直接缓存数据，query_cache_type 是否开启缓存，这个需要修改配置文件
  - query_cache_size 缓存大小。`set GLOBAL query_cache_size = 1024000`

## 存储引擎

我们现在最常用的存储引擎是 innodb 和 myisam，然后还有很多其他引擎，可以通过 `show engines`查看支持哪一些引擎，甚至可以安装引擎，因为 mysql 的存储引擎是插件化的

先来说一下怎创建不同存储引擎的表，其实就是在 engine 上填上不同的存储引擎名称

```sql
CREATE TABLE `myisamt` (
  `id` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8mb4;
```

### myisam

data 文件，`show global variables like 'datadir'` 可以查看到数据文件的位置，myisam 存储引擎的表会有三个文件，具体如下

```
// 这里我的数据库名称是 test，所以在 test 路径下
# pwd
/var/lib/mysql/test

# tree
├── myisamt.frm     # 存储表结构 任何存储引擎都会有
├── myisamt.MYD   # 数据库文件
└── myisamt.MYI     # 索引文件
```

myisam 特性

- 并发特性：表级锁
- 支持全文搜索（后来 innodb 也支持了）
- 支持索引数据压缩
  - 使用 myisampack 工具进行压缩
  - myisampack -b -f xxx.myi

myisam 适用非事务、只读、空间类应用（后来 mysql5.7 innodb 也支持空间索引了～）

### innodb

data 文件

```
├── innodbt.frm     # 存储表结构
├── innodbt.ibd     # 数据+索引文件
```

特性

- 事务性。ACID
- 行级锁
- Redo log 和 undo log，提交事务的时候要把事务日志写到这里

跟 myisam 区别
- innodb 行级锁，并发性更高
- innodb有外键和事务
- innodb有系统表空间和独立表空间
- myisam不能缓存真实的数据，只缓存索引

所以什么是系统表空间和独立表空间呢。innodb_file_per_table 默认开启，表的数据文件和索引会单独一个文件，系统表空间只会创建一个 frm 文件存储表结构，数据和索引全部存到统一的一个文件，很容易想到这样 io 瓶颈会有问题，推荐是使用独立表空间的，默认也是独立表空间

### csv

data 文件

```
├── csvt.CSM   # 表的元数据，状态信息和数据量
├── csvt.CSV    # 数据文件
├── csvt.frm     # 表结构
```

特性

- csv 格式存储，可以直接编辑
- 所有列不能为null
- 不支持索引

### archive

data 文件

```
├── archivet.ARZ    # 数据文件
├── archivet.frm     # 表结构
```

特性

- 以zlib 对数据进行压缩，磁盘IO更少
- 只支持 insert 和 select
- 只允许自增ID列上加索引

使用场景 日志和数据采集应用

### memory

没有 data 文件，存在内存中

特性

- 数据存在内存里
- 支持HASH索引和BTree索引
- 所有字段固定长度，不支持BLOG和Text 大字段
- 表级锁
- 最大大小 max_heap_table_size，设置容量

memory 和临时表是不一样的，临时表分为

- 系统创建临时表。系统在执行 sql 语句时，有时候会创建临时表，当表没有超过容量限制的时候，创建的是 memory 表；当超过容量限制的时候，创建的是 myisam 表
- 用户创建临时表，使用 `create temporary table` 这种语法去创建，即在正常创建表的前面加 temporary

### ferderated

远程访问的表，本地实际没有数据，数据都在远程，本地只保存表结构和远程连接信息

特性

- 远程访问mysql服务器上的表
- 本地不存储数据，所有数据都在远程
- 本地需要保存表结构和服务器连接信息

使用场景：偶尔的统计分析和手工查询

由于时间的关系，创建的语法这里就直接从 mysql 官方文档贴创建语句了，其实创建的时候多加一个远程连接的信息就可以了

```
CREATE TABLE `T1`(
  `A` VARCHAR(100),UNIQUE KEY(`A`(30))
  ) ENGINE=FEDERATED
  CONNECTION='MYSQL://127.0.0.1:3306/TEST/T1';
```

但是 ferderated 引擎默认是不开启的，需要修改配置文件，在配置文件上加 ferderated=1。这个大家可以在网上查找一下怎么在 mysql 配置文件上加这个选项

今天我们聊了一下 mysql 大概的逻辑架构，监控 tps 和 qps，最后聊了一下存储引擎。这里只是非常浅了聊了一下，mysql 的内容非常丰富，深入的了解就需要去 [mysql官方文档](https://dev.mysql.com/doc/refman/5.7/en/preface.html) 或者阅读书籍了