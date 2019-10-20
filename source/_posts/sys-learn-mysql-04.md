---
title: 【系统学习-mysql】【四】索引和执行计划
categories:
  - mysql 学习
tags:
  - mysql
date: 2019-10-19 16:23:04
---

## 概要

来了，它来了。聊着聊着就聊到了要怎么优化sql的事情上了，sql优化我们很大程度上都是在分析sql执行计划和使用索引上搞事情，毕竟索引是专门用来提高查询效率的数据结构。正如大家常年举的新华字典例子，索引就想字典的目录一样，是用来帮助快速查找的

那么在真实生产环境中，我们优化查询的步骤是先找到慢查询，然后分析sql执行计划，最后在索引上搞文章。当然，一切的一切，表的设计要正确，只有表设计好了，才有sql优化发挥的空间，只是我们这里不聊表的设计。所以我们今天聊的有以下几点：

- 慢查询
- 索引
- 执行计划
- 常见的索引优化

<!-- more -->

## 慢查询

慢查询是指执行查过一定时间阈值的sql语句，一般情况下，我们分析慢查询就是指分析mysql中的慢查询日志，慢查询日志是我们找到需要做sql优化的基础。但是默认情况下慢查询日志是关闭的，如果我们不是使用云服务提供mysql数据库，那么我们需要先开启这个日志。在开启之前，我们先聊几个有关慢查询的参数配置

- slow_query_log 启动停止慢查询日志
- slow_query_log_file 指定慢查询日志的存储路径及文件（默认和数据文件放一起）
- long_query_time 指定记录慢查询日志SQL执行时间得阈值（单位：秒，默认10秒）
- log_queries_not_using_indexes  是否记录未使用索引的SQL
- log_output 日志存放的地方【TABLE】【FILE】【FILE,TABLE】**生产环境使用默认的 FILE就好了**

具体开启日志的方式，直接修改 `/etc/mysql/mysql.conf.d/mysqld.cnf` 或者 `/etc/mysql/my.cnf` 文件(注意不同系统或者不同版本位置不一样)，添加以下几行

```
# 注意如果已经有 [mysqld] 了，就不需要添加 [mysqld] 了
[mysqld]
slow_query_log          = 1
slow_query_log_file     = /var/log/mysql/mysql-slow.log
# 注意这个时间需要根据自己的需求进行修改
long_query_time = 2
log-queries-not-using-indexes
```

当我们开启了慢查询日志的时候，符合慢查询条件的sql语句就会被记录到慢查询日志中，sql语句包括`查询语句、数据修改语句、已经回滚的Sql`。那么我们可以来看一下慢查询日志是什么亚子的，这里建议大家实际操作一下

```
# Time: 2019-10-19T10:59:25.885136Z
# User@Host: root[root] @ localhost [127.0.0.1]  Id:     3
# Query_time: 0.000074  Lock_time: 0.000032 Rows_sent: 1  Rows_examined: 1
SET timestamp=1571482765;
SHOW INDEX FROM `test`.`innodbt`;
```

没一条慢查询日志大概就跟上面这个一样，第一行是查询的时间，第二行是用户名、用户ip、线程id，第三行是查询花费的时间、获得锁的时间、查询的结果行数、扫描的行数，第四行是执行sql的具体时间，第五行是执行的sql语句

### 分析慢查询的工具

通过查看慢查询日志，我们可以得到sql语句的执行时间，用户信息等等，但是直接看就太不方便了，所以我们需要借助工具来分析。这里有两个工具，一个是mysql官方自带的 `mysqldumpslow`，另外一个是 `pt-query-digest`。我们先来看一下这两个工具是怎么使用的，可以为我们获取到什么样的信息，由于我直接在我本地开启的，就没有什么慢查询的语句了

```
$ sudo mysqldumpslow -s t -t 10 mysql-slow.log

Reading mysql slow query log from mysql-slow.log
Count: 6  Time=0.00s (0s)  Lock=0.00s (0s)  Rows=0.8 (5), root[root]@localhost
  show variables like 'S'

Count: 11  Time=0.00s (0s)  Lock=0.00s (0s)  Rows=1.0 (11), root[root]@localhost
  SHOW SESSION VARIABLES LIKE 'S'

Count: 2  Time=0.00s (0s)  Lock=0.00s (0s)  Rows=2.0 (4), root[root]@localhost
  SHOW COLUMNS FROM `test`.`account`
```

上面的是使用 mysqldumpslow 进行分析的，统计了语句具体语句执行的次数等相对较简单的信息。稍微介绍一下参数的意思，`-s`是指按什么排序的意思，`t`是按照总时间进行排序，`-t 10`是指取前10天作为输出结果。更多的参数可以 `man mysqldumpslow` 进行查看

接下来我们看一下 pt_query_digest，首先下载地址 [pt-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-query-digest.html#downloading)，同样这个链接也是使用手册的地址。安装好后我们来试一下

```
$ sudo pt-query-digest --explain h=127.0.0.1,u=root,p=123456 mysql-slow.log

# 140ms user time, 20ms system time, 39.86M rss, 121.93M vsz
# Current date: Sat Oct 19 20:32:00 2019
# Hostname: sealde-MS-7B23
# Files: mysql-slow.log
# Overall: 297 total, 59 unique, 0.04 QPS, 0.00x concurrency _____________
# Time range: 2019-10-19T10:40:42 to 2019-10-19T12:32:00
# Attribute          total     min     max     avg     95%  stddev  median
# ============     ======= ======= ======= ======= ======= ======= =======
# Exec time           74ms     1us     6ms   247us   799us   601us    73us
# Lock time           13ms       0   362us    42us   144us    62us    17us
# Rows sent            943       0     499    3.18    2.90   30.80       0
# Rows examine      25.31k       0   1.00k   87.26 1012.63  274.01       0
# Query size        14.67k       0     215   50.56  158.58   43.94   30.19

# Profile
# Rank Query ID                        Response time Calls R/Call V/M   It
# ==== =============================== ============= ===== ====== ===== ==
#    1 0xE77769C62EF669AA7DD5F6760F...  0.0223 30.4%    11 0.0020  0.00 SHOW VARIABLES
#    2 0x873D3AAF7C4528CF9439C8E2DE...  0.0144 19.6%    11 0.0013  0.00 SHOW VARIABLES
#    3 0x6B18859E0CFC8086C0F01DC5D4...  0.0042  5.7%    17 0.0002  0.00 SELECT performance_schema.events_statements_current performance_schema.threads
#    4 0xE8390778DC20D4CC04FE01C5B3...  0.0040  5.5%   100 0.0000  0.00 ADMIN PING
#    5 0x3FDBE6B6A9945C2923C20C3122...  0.0029  3.9%     2 0.0014  0.00 SHOW COLUMNS
#    6 0x622EF34FA79B543FDC5AC4F51E...  0.0027  3.6%     1 0.0027  0.00 SHOW GLOBAL VARIABLES
#    7 0xED814B5BEEF0BCD4445D653901...  0.0021  2.8%    11 0.0002  0.00 SELECT innodbt
#    8 0xA12763007BC796F0F218D206B0...  0.0018  2.5%    17 0.0001  0.00 SELECT performance_schema.events_waits_history_long
#    9 0x7E4FD8B840A40A1DF6AD3D55D5...  0.0018  2.4%    17 0.0001  0.00 SELECT performance_schema.events_stages_history_long
#   10 0x54D305756DFCDBFA06EB5CA98D...  0.0015  2.0%    12 0.0001  0.00 SHOW INDEX
#   11 0xCB35558F37B42E38A2750AF24E...  0.0011  1.4%     1 0.0011  0.00 SHOW COLLATION
#   12 0xF4182D1E505F5242FBC250C228...  0.0011  1.4%     6 0.0002  0.00 SHOW TRIGGERS
#   13 0xA39DD594FAE10E151F9C4ED1B9...  0.0010  1.3%     1 0.0010  0.00 SHOW FUNCTION STATUS
#   14 0x507A8C3C929392C00CD32C60BE...  0.0009  1.2%     1 0.0009  0.00 SHOW PROCEDURE STATUS
#   15 0x37C662AFC001EDCC242B6C923E...  0.0007  1.0%     1 0.0007  0.00 SHOW COLUMNS
#   16 0x96B4D2F0736877374BAEF5D5D4...  0.0007  0.9%     1 0.0007  0.00 SHOW COLUMNS
#   17 0x0C9C70920DA7300E0B14E77CA2...  0.0007  0.9%     2 0.0003  0.00 SHOW STATUS
#   18 0x751417D45B8E80EE5CBA203445...  0.0007  0.9%     2 0.0003  0.00 SHOW DATABASES
#   19 0x0C86A6D1EBA3C4420E346EA322...  0.0007  0.9%     1 0.0007  0.00 SHOW COLUMNS
#   20 0x9BB432DAD869152599B703A34B...  0.0007  0.9%     1 0.0007  0.00 SHOW COLUMNS
# MISC 0xMISC                           0.0078 10.6%    81 0.0001   0.0 <39 ITEMS>

# Query 1: 0.00 QPS, 0.00x concurrency, ID 0xE77769C62EF669AA7DD5F6760F2D2EBB at byte 1256
# This item is included in the report because it matches --limit.
# Scores: V/M = 0.00
# Time range: 2019-10-19T10:41:01 to 2019-10-19T12:32:00
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count          3      11
# Exec time     30    22ms   754us     6ms     2ms     5ms     2ms   761us
# Lock time     12     2ms    65us   338us   143us   316us    98us    76us
# Rows sent      1      10       0       1    0.91    0.99    0.29    0.99
# Rows examine  43  11.02k   1.00k   1.00k   1.00k   1.00k       0   1.00k
# Query size     2     402      35      41   36.55   40.45    2.11   34.95
# String:
# Databases    test
# Hosts        localhost
# Users        root
# Query_time distribution
#   1us
#  10us
# 100us  ################################################################
#   1ms  #####################################################
#  10ms
# 100ms
#    1s
#  10s+
show variables like 'slow_query_log'\G

...

SELECT st.* FROM performance_schema.events_statements_current st JOIN performance_schema.threads thr ON thr.thread_id = st.thread_id WHERE thr.processlist_id = 4\G
# *************************** 1. row ***************************
#            id: 1
#   select_type: SIMPLE
#         table: thr
#    partitions: NULL
#          type: ALL
# possible_keys: NULL
#           key: NULL
#       key_len: NULL
#           ref: NULL
#          rows: 256
#      filtered: 10.00
#         Extra: Using where
# *************************** 2. row ***************************
#            id: 1
#   select_type: SIMPLE
#         table: st
#    partitions: NULL
#          type: ALL
# possible_keys: NULL
#           key: NULL
#       key_len: NULL
#           ref: NULL
#          rows: 2560
#      filtered: 10.00
#         Extra: Using where; Using join buffer (Block Nested Loop)
```

可以看到上面有三段，第一段是汇总信息，包括执行时间、获取锁的时间、获取行数、扫描行数、查询大小；第二段根据执行的响应时间倒叙排列，包含的信息有响应时间、执行次数、平均响应时间(总响应时间/执行次数)、大致的查询语句；第三段就是具体的执行情况了，其实按正常的语句来说，是可以得到执行计划的，类似于第三段的最后一部分

总的来说两者都能分析慢查询，但功能和获得的信息来说 `pt-query-digest` 更加优秀。当然工具还有很多，这里也只是聊一下工具可以帮我们做些什么

## 索引

在分析执行计划前，我们先来聊一下索引。首先索引是帮助 mysql 高效获取数据的数据结构，它的本质就是数据结构，mysql索引的数据结构有 `B-Tree`、`R-Tree`、`Hash`、`Fulltext`。我们最长用的索引就是 `B-Tree` 这种数据结构的索引，`R-Tree` 是空间索引，`Hash` 索引是只支持精确匹配的，而且没有排序，`Fulltext` 全文索引适用于全文检索

通过将主键和索引字段以及真实数据的地址，写入到索引数据结构中，利用数据结构的查询优势，来提高数据的查询效率。很显然，索引会影响数据的插入、更新和删除的速度，因为需要额外的开销去维护索引

### 二叉树、B-Tree、B+Tree

在这里大概聊一下这几个数据结构，首先二叉树是一种有序的数据结构，左边子节点的数比当前节点的数小，右边子节点的数比当前节点的数大，当要查找一个数的时候，顺着节点查找，如果小于节点的数就往左边查找，大于节点的数就往右边查找，有效地减少了访问数据的次数。如下图是一张二叉树的图

{% asset_img binary-tree.png %}

但是我们也发现，树可能会往一边倾斜，这样子查询的效率就会变低，极端情况时间复杂度是 O(n)。这时候，就出现了**平衡二叉树**，平衡二叉树会在每次插入或删除数据的时候，检查树是否倾斜，然后调整树的平衡。可以在[算法可视化](https://www.cs.usfca.edu/~galles/visualization/Algorithms.html)这个教学网站上找一个平衡二叉树感受一下

B-Tree 和 B+Tree 都是平衡树，我们先来看一下这两个树的结构

{% asset_img b-tree.jpeg %}

B-Tree 也叫平衡多路查找树，跟上面的二叉树不同的是，它的一个节点可以有多个数据。由于计算机读取磁盘是整块读取的关系，这种数据结构就很适合数据库。我们再来看一下 B+Tree 有什么不一样

{% asset_img b+tree.jpeg %}

可以看到，B+Tree 跟 B-Tree 唯一的不同就是所有数据都会落在叶子节点上(最下面一层的节点)。innodb 的 B-Tree 索引使用的就是 B+Tree，而且把叶子节点连起来了，并且非叶子节点只存key。由于计算机读取磁盘是按整块读的，innodb 自己也有最小的存储单位，默认是 16k，称为**一页**，可以通过执行 `show variables like 'innodb_page_size'` 查看，结合以上我们可以得出，使用 B+Tree 有以下几点优势

- 减少io读取次数，因为能更好地利用计算机读取磁盘的特性
- 比起 B-Tree 来说，更适合范围查找。因为将叶子节点连起来了，B-Tree 的话需要做更复杂的遍历
- 对比 B-Tree 来说，非叶子节点存储的数据大小更小，意味着io读取次数可能更少。因为 B+Tree 非叶子节点只存key值，数据大小可能可以一次读取，B-Tree的话可能会读取两次

### 索引的分类

索引按使用来分，可以分为以下几种：

- 普通索引：即一个索引只包含单个列，一个表可以有多个单列索引
- 唯一索引：索引列的值必须唯一，但允许有空值
- 复合索引：即一个索引包含多个列

索引还可以分为

- 聚集索引(聚簇索引)：数据的物理顺序和索引列的顺序一致。可以更好地按页读取数据，减少io操作。一张表只有一个聚集索引，一般为主键。可以阅读 [mysql5.7 Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html) 了解一下
- 非聚集索引：不是聚集索引的索引，所以一般我们创建的索引属于这一种

## 执行计划

这个系列第一篇聊过，sql语句在经过语法解析后，会交给优化器进行优化，然后再执行。而执行计划是可以模拟优化器执行sql查询语句从而知道MySQL是如何处理你的SQL语句的，分析你的查询语句或是表结构的性能瓶颈。语法 `explain + sql语句`

```sql
explain select * from innodbt;
```

| id | select\_type | table | partitions | type | possible\_keys | key | key\_len | ref | rows | filtered | Extra |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| '1' | 'SIMPLE' | 'innodbt' | NULL | 'ALL' | NULL | NULL | NULL | NULL | '1' | '100.00' | NULL |

接下来一起聊一下这些都代表什么意思

### id

第一列是 id，表示查询中子句执行的顺序。顺序的原则是id相同，由上至下执行；id越大优先级越高，越先执行。比如下面的执行顺序会是 第三行 -》第一行 -》 第二行

| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| '1' | 'SIMPLE' | 'i' | NULL | 'ALL' | 'PRIMARY' | NULL | NULL | NULL | '1' | '100.00' | NULL |
| '1' | 'SIMPLE' | 'a' | NULL | 'eq_ref' | 'PRIMARY' | 'PRIMARY' | '4' | 'test.i.id' | '1' | '33.33' | 'Using where' |
| '2' | 'SIMPLE' | 'm' | NULL | 'eq_ref' | 'PRIMARY' | 'PRIMARY' | '4' | 'test.i.id' | '1' | '100.00' | 'Using index' |

### select_type

查询的类型，作用是用于区别普通查询、联合查询、子查询等的复杂查询。一共有以下几种类型

- SIMPLE 简单查询。查询中不包含子查询或者UNION
- PRIMARY 最外层查询。查询中若包含任何复杂的子部分，最外层查询则被标记为
- SUBQUERY 子查询。在SELECT或WHERE列表中包含了子查询
- DERIVED 衍生查询。在FROM列表中包含的子查询被标记为DERIVED(衍生)MySQL会递归执行这些子查询, 把结果放在临时表里
- UNION union的第二个select语句
- UNION RESULT 从UNION表获取结果的SELECT

```sql
-- 子查询
explain select t1.*,(select t2.id from myisamt t2 where t2.id = 1) from innodbt t1;
```

| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| '1' | 'PRIMARY' | 't1' | NULL | 'ALL' | NULL | NULL | NULL | NULL | '1' | '100.00' | NULL |
| '2' | 'SUBQUERY' | 't2' | NULL | 'const' | 'PRIMARY' | 'PRIMARY' | '4' | 'const' | '1' | '100.00' | 'Using index' |

```sql
-- 衍生查询
explain select t1.* from innodbt t1, (select min(id) as s from myisamt t2) s2 where t1.id = s2.s;
```

| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| '1' | 'PRIMARY' | '<derived2>' | NULL | 'system' | NULL | NULL | NULL | NULL | '1' | '100.00' | NULL |
| '1' | 'PRIMARY' | 't1' | NULL | 'const' | 'PRIMARY' | 'PRIMARY' | '4' | 'const' | '1' | '100.00' | NULL |
| '2' | 'DERIVED' | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | 'Select tables optimized away' |

### table

显示这一行的数据是关于哪张表的

### type

**重要的参考数据**，type表示访问类型，结果值从最好到最坏依次是 `system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index> ALL `。但我们真正需要记住的是这个顺序 `system>const>eq_ref>ref>range>index>ALL`，我们的目标是达到 range 和 ref

- system 表只有一行记录（等于系统表）
- const 索引一次就找到了。比如主键或唯一索引
- eq_ref 主键或唯一索引扫描。比如 `select * from t1, t2 where t1.id = t2.id`
- ref 意味着使用到了索引。非唯一性索引扫描，返回匹配某个单独值的所有行
- range between、<、>、in 范围查询在索引上
- index **这个不是使用到索引**。查询的结果全为索引列的时候，这种也是全表扫描索引文件
- all 扫描全表

### possible_key & key

**possible_key 是指可能会使用的索引；key 是指实际上使用到的索引**

有时候由于 sql 语句问题，可能会使用索引，但实际上没有使用到索引，就会出现 possible_key 有值而 key 为 null；但也有时候，可能不会使用到索引，但实际上又使用到了索引，就会出现 possible_key 为 null，而 key 有值

### key_len

key_len 表示索引中使用的字节数，可通过该列计算查询中使用的索引长度

- 在不损失精确性的情况下，长度越短越好。因为索引占用字节数短有助于节省空间和提高索引的搜索效率
- 在使用复合索引的时候，key_len 可以判断所有的索引字段是否都被查询用到。这个时候，长度越长越好，意味着使用到的索引越多，这个我们晚点再聊

索引的长度跟索引字段的数据类型有关系，跟是否为 null 有关系，跟是否变长有关系，还跟字符的编码有关系，我们可以通过计算索引的长度(key_len)来分析使用索引的情况。我们先来看一下有哪些长度规则，然后再看几个例子

- 字符类型的长度跟字符编码有关系，latin占1个字节，gbk占2个字节，utf-8占3个字节，utf8mb4占4个字节
- 变长数据类型需要 2 个字节维护
- 数据类型是否为 null 需要 1 个字节维护
- 复合类型的 key_len 等于使用到的索引长度之合

接下来我们看一下例子，结合下面的例子，不难得出刚刚的结论

```sql
CREATE TABLE `idx_test` (
  `id` int(11) NOT NULL,
  `name` char(11) DEFAULT NULL,
  `created_at` datetime NOT NULL,
  `phone` char(11) NOT NULL,
  `desc` varchar(11) DEFAULT NULL,
  `c_1` int(2) NOT NULL,
  `c_2` char(2) NOT NULL,
  `c_3` datetime NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`),
  KEY `idx_createdAt` (`created_at`),
  KEY `idx_phone` (`phone`),
  KEY `idx_desc` (`desc`),
  KEY `idx_c123` (`c_1`,`c_2`,`c_3`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- key_len 为 44；44 = 4 * 11。utf8mb4 一个字符占4个字节
explain select * from idx_test where phone = 'a';
-- key_len 为 45；45 = 4 * 11 + 1。因为 name 不是 not null 的，所以需要多 1 个字节维护
explain select * from idx_test where name = 'a';
-- key_len 为 47；47 = 4 * 11 + 1 + 2。因为 desc 不是 not null 需要多 1 个字节维护，并且是变长的需要再多 2 个字节维护
explain select * from idx_test where `desc` = 'a';
-- key_len 为 5；datetime 类型为 5 个字节
explain select * from idx_test where created_at = '2019-10-22';

-- key_len 为 17;17 = 4 + 4*2 + 5。复合索引的索引长度为使用到索引的和
explain select * from idx_test where c_1 = 'a' and c_2 = 'b' and c_3 = 'c';
-- key_len 为 12;12 = 4 + 4*2。这里使用到了 c_1 和 c_2 两个字段
explain select * from idx_test where c_1 = 'a' and c_2 = 'b';
```

我们再聊一下复合索引，复合索引有一个原则，叫最左前缀原则，即建立索引的顺序(比如 c_1，c_2，c_3)，在使用到的索引字段没有最左边顺序的索引，这个复合索引会失效；并且复合索引中，使用范围查询，会让后面的索引字段失效，比如 c_1 使用范围查询，c_2、c_3 就利用不到索引。这不是很好解释，我们来看例子

```sql
-- 这条 sql 没有使用到索引，因为没有使用到 c_1，后面的字段即使用到，索引也失效了
explain select * from idx_test where c_2 = 'b' and c_3 = 'c';
-- key_len 为 4，这说明只使用到 c_1 字段，c_3 并没有使用到，因为中间 c_2 断了
explain select * from idx_test where c_1 = 'a' and c_3 = 'c';
-- key_len 为 12，这说明字段的使用顺序没有什么影响，只要使用到的字段是从左往右的
explain select * from idx_test where c_2 = 'b' and c_1 = 'a';
-- key_len 为 4，这说明复合索引使用范围查询的时候，会使后面的索引字段失效
explain select * from idx_test where c_2 = 'b' and c_3 = '2019-10-22' and c_1 > 2;
```

### ref

ref 显示索引的哪一列被使用了，有可能是一个常量值。下面执行 `explain select * from idx_test where c_1 = 'a' and c_2 = 'b'`

| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |
| '1' | 'SIMPLE' | 'idx_test' | NULL | 'ref' | 'idx_c123' | 'idx_c123' | '12' | 'const,const' | '1' | '100.00' | NULL |

### rows

row 大致估算出找到所需的记录所需要读取的行数

### Extra

这也是**重要参考参数**，Extra 表示额外信息，常见的额外信息包括

- Using filesort：没有使用索引排序，而是按照外部排序
- Using temporary：使用了临时表，常见于 order by 和 group by
- Using index：是否用了覆盖索引
- Using where：使用了 where 过滤
- Using join buffer：使用了连接缓存
- Impossible where：不能得到数据的条件

其中 Using filesort 和 Using temporary 是可以做优化的，目标是不让这两个信息出现在 Extra 中。我们来看两个例子，这里有一个很关键的理解，**覆盖索引是指数据可以直接从索引文件或索引的数据结构中获取，而不需要做再次查询获取额外数据**

第一个例子，当 order by 的时候，如果使用了索引字段意外的字段，Extra 则为 filesort；相反，则是使用了覆盖索引，覆盖索引的效率比 filesort 高，需要注意的是主键 id 也在索引数据结构中

```sql
-- Extra 为 Using filesort
explain select * from idx_test i order by name;
-- Extra 为 Using index
explain select name, id from idx_test i order by name;
```

第二个例子，第二条语句跟第一条语句唯一的不同就是 group by 多了一个 c_1 字段，因为 c_1,c_2,c_3 组成了复合索引，由于最左前缀原则，第一条语句没有使用到索引。当然实际生产中需要结合业务进行这样的 sql 优化，比如调整一下索引的顺序

```sql
-- Extra 为 Using where; Using temporary; Using filesort
explain select c_1, c_2, c_3, name from idx_test i where c_2 in ('a', 'b') group by c_2;
-- Extra 为 Using where
explain select c_1, c_2, c_3, name from idx_test i where c_2 in ('a', 'b') group by c_1, c_2;
```

## 常见的sql优化

在了解了索引和执行计划信息之后，我们就可以来看看 sql 优化常用的策略

- 尽量全值匹配。当使用复合索引的时候，尽量使用到全部字段(即 key_len 达到最大)
- 最佳左前缀匹配。复合索引中从最左边的列开始，并且不跳过中间的列
- 不在索引上做任何操作，比如计算，or，函数等会使索引失效。因为这样会使索引失效
  - `explain select * from idx_test i where left(phone, 1) = 'a'` left 函数会让索引失效
- 复合索引中，需要返回查询的列放在最后。因为范围查询会让后面的列失效，上面有例子
- 覆盖索引尽量使用，少用 select *
  - `explain select c_1, c_2, c_3, id from idx_test where c_1 = 'a'` Extra 会使用 Using index，而 `explain select * from idx_test where c_1 = 'a'` 则不会
- 不等于要慎用
  - `explain select * from idx_test i where phone != 'a'` 索引将失效
  - 可以通过使用覆盖索引提高效率 `explain select phone from idx_test i where phone != 'a'`
- null/not null 会影响索引使用情况。当列是 not null 时(phone)，查询`where phone is null`，直接 impossible where；查询`select * from idx_test where phone is not null` 时，索引失效；当列是 null 时(name)，查询`where name is null`，会使用索引；查询`select * from idx_test where name is not null` 时，索引失效
  - 可以通过覆盖索引提高效率。`explain select name from idx_test i where name is null` 索引生效
- like 要注意。当使用左通配符或左右统配符时，索引失效。比如 `like '%a'` 或 `like '%a%'`
- 字符类型加引号。不加引号，索引会失效，因为数据进行了转换
- or 改 union 效率提高。or 会让索引失效，使用 union 的话可以让索引生效，只是这个不好操作

### 批量导入优化

在批量导入数据的时候，也可以进行优化。如果是一条数据一条数据导入，速度会非常慢，我们可以优化的操作有

- 提交前关闭自动提交。比如 `conn.setAutoCommitted(false)`
- 尽量使用批量 sql 语句。即将 sql 拼成 `insert into table (xxx,xxx) values (yyy,yyy),(zzz,zzz)...;`
- 使用 myisam 存储引擎导入，虽然我一般不这么干，太麻烦

更快的方法是通过文件进行导入，速度会比执行 sql 语句快 10 几 20倍。load_data_file

```
-- 将数据到处到文件
select * into OUTFILE 'xxx文件路径' from table_name
-- 将文件数据导入到表
load data INFILE 'xxx文件路径' into table table_name
```

那么我们 mysql 系列就聊到这里了，我们从 Mysql 的逻辑架构存储引擎聊到了锁，再聊到事务，最后以索引和 sql 优化结束，相信大家对 mysql 有了一个稍微全面一点的认识。我们还有很多没有聊到的，比如主从复制啦，存储引擎更多的特性啦，事务日志(重做redo和回滚undo)啦，主从一致性校验啦等等。这些大家有兴趣都可以慢慢去了解，好了，下次再聊了～