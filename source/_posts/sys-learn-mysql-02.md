---
title: 【系统学习-mysql】【二】锁
categories:
  - mysql 学习
tags:
  - mysql
date: 2019-10-17 01:00:00
---

## 概要

最近白昼变短，我们也就聊少一点，今天就来聊一下 Mysql 中的锁。其实 Mysql 中锁的概念跟我们之前聊过 java 里面的锁是很像的，共享锁和独占锁，共享锁是可以有多个线程占有的，独占锁是只有一个线程占有，也叫排它锁。当然还有其他锁，比如意向锁、gap锁、next-key锁等等，但是我们现在只聊最常用的。所以我们今天来聊聊以下几个方面

- 锁分类
- 表的读锁和写锁
- 行的共享锁(S)和独占锁(X)
- 常见问题

<!-- more -->

## 锁分类

Mysql 锁从粒度来分，可以分为 表锁、行锁、页面锁，比如 myisam 存储引擎只支持表锁，而 innodb 支持行锁。所谓表锁就是对整张表进行加锁，行锁是对一行数据进行加锁，类似于 Java 里面锁一个类(虽然锁的是类对象)和锁一个对象。不同粒度的锁性能，使用场景上会有锁差别

- 表锁
  - 开销小，加锁快
  - 不会死锁，锁粒度大，冲突高，并发低
- 行锁
  - 开销大，加锁慢
  - 会死锁，粒度小，冲突低，并发高
- 页面锁
  - 开销和冲突介于表级锁和行级锁之间
  - 会死锁

我们再来看一下加锁的语法

```sql
-- 锁表
lock table 表名 read/write;
-- 解锁表
unlock tables;

-- 行锁需要先开启事务，即 begin; 提交事务后，会释放锁，即 commit/rollback
-- 共享式锁行
select * from 表名 where 条件 lock in share mode
-- 独占式锁行
select * from 表名 where 条件 for update
```

官方文档[Mysql5.7表锁语法](https://dev.mysql.com/doc/refman/5.7/en/lock-tables.html)。当然 mysql 的锁还有好多种，比如 意向锁、记录所、gap锁、next-key锁、插入意向锁等，我们现在只聊常用的。详细可以参考一下 [Mysql5.7 InnoDB Locking](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)

## 表的读锁和写锁

其实 Mysql 的读锁跟共享锁是一样样的，写锁跟独占锁（或者叫排它锁）是一样样的。我们接下来先看看锁之间有什么相互作用，然后再进行相应的测试进行验证，希望大家也做一般验证的操作，这会加深对 mysql 锁的理解

| 锁类型 | 同session同表 | 同session不同表 | 不同session同表 | 不同session不同表 |
| :-------: | :-------: | :-------: | :-------: | :-------: |
| 读锁     | 可以查询/不可以修改 | 什么操作都不可以 | 可以查询/修改等待 | 可以做任何操作 |
| 写锁     | 可以查询修改 | 什么操作都不可以 | 查询等待/修改等待 | 可以做任何操作 |

大致来说，加读锁(共享锁)，其他session可以读，不可以写；同一session可以读，不可以写。加写锁(排它锁)，其他session不可以读，不可以写；同一session可以读，也可以写。但不管读锁还是写锁，同一session不能对其他表做任何操作，包括查询

### 表加读锁

sessionA 加写锁

```sql
lock table test.myisamt read;
-- 对表加读锁时，不能对数据进行更新。直接报错
INSERT INTO `test`.`myisamt`(`id`,`name`)VALUES(2,'aa');
update test.myisamt set name = 'b' where id = 1;
delete from test.myisamt where id = 1;
-- 对表加读锁时，可以对数据进行查询
select * from test.myisamt;

-- 对表加读锁时，对其他表不能做任何操作(包括修改表结构)，直接报错
select * from test.innodbt;
insert into test.innodbt (`id`) values (1);

unlock tables;
```

sessionB

```sql
-- 不同session。这里有另外一个session对myisamt加读锁
-- 可以查询
SELECT * FROM test.myisamt;
-- 不能对数据进行更新，进入阻塞状态
INSERT INTO `test`.`myisamt`(`id`,`name`)VALUES(3,'aaa');
update test.myisamt set name = 'b' where id = 1;
delete from test.myisamt where id = 1;

-- 对其他表可以做任何操作
select * from test.innodbt;
insert into test.innodbt (`id`, `name`) values (2, 'abc');
```

### 表加写锁

sessionA 加写锁

```sql
lock table test.myisamt write;
-- 对表加写锁时，可以对数据进行更新
INSERT INTO `test`.`myisamt`(`id`,`name`)VALUES(2,'aa');
update test.myisamt set name = 'b' where id = 2;
delete from test.myisamt where id = 1;
-- 对表加写锁时，可以对数据进行查询
select * from test.myisamt;

-- 对表加读锁时，对其他表不能做任何操作(包括修改表结构)，直接报错
select * from test.innodbt;
insert into test.innodbt (`id`) values (1);

unlock tables;
```

sessionB

```sql
-- 不同session。这里有另外一个session对myisamt加写锁
-- 不可以查询，进入阻塞
SELECT * FROM test.myisamt;
-- 不能对数据进行更新，进入阻塞状态
INSERT INTO `test`.`myisamt`(`id`,`name`)VALUES(4,'aaa');
update test.myisamt set name = 'b' where id = 1;
delete from test.myisamt where id = 1;

-- 对其他表可以做任何操作
select * from test.innodbt;
insert into test.innodbt (`id`, `name`) values (3, 'abc');
```

## 行的共享锁(S)和独占锁(X)

innodb 即支持行锁也支持表锁，表锁几乎跟 myisam 的表锁一样，表锁的语法是一样的。**锁行需要先开启事务，并且如果重新开启事务，锁会释放**。

- 行加独占锁(x)时，该事务下可以对数据进行任意的操作；其他事务可以查询，但对锁住的数据更新、获取共享锁、获取独占锁将进入阻塞
- 行加共享锁(s)时，该事务下可以对数据进行任意的操作；其他事务可以查询、获取共享锁，但对锁住的数据更新、获取独占锁将进入阻塞

总的来说，独占锁不能有多个事务同时拥有，共享锁可以；独占锁和共享锁对行数据加锁时，其他事务不能更新该行数据，其他数据可以进行操作。特别地，如果只有事务A加了共享锁，事务A内可以对该数据更新；有其他事务B也加了共享锁，事务A更新该行数据经进入阻塞

### 行加共享锁

事务A

```sql
use test;

begin;
-- 行加共享锁时
select * from test.innodbt where id = 2 lock in share mode;
-- 可以对加锁的行做任何操作
select * from test.innodbt;
update test.innodbt set name = 'aaa' where id = 2;
delete from test.innodbt where id = 2;

-- 可以对非加锁的行做任何操作
INSERT INTO `test`.`innodbt`(`id`,`name`)VALUES(4,'test');
update test.innodbt set name = 'ttt' where id = 4;
delete from test.innodbt where id = 4;

-- 可以对其他表做任何操作
select * from test.myisamt;
INSERT INTO `test`.`myisamt`(`id`,`name`)VALUES(9,'aaa');

rollback;
```

事务B

```sql
use test;

-- 另外一个事务加独占锁时
begin;
-- 可以查询
select * from test.innodbt;
-- 更新进入阻塞
update test.innodbt set name = 'popp' where id = 2;

-- 获取独占锁进入阻塞
select * from test.innodbt where id = 2 for update;
-- 获取共享锁成功
select * from test.innodbt where id = 2 lock in share mode;
rollback;
```

### 行加独占锁

事务A

```sql
use test;

begin;
-- 行加独占锁时
select * from test.innodbt where id = 2 for update;
-- 可以对加锁的行做任何操作
select * from test.innodbt;
update test.innodbt set name = 'aaa' where id = 2;
delete from test.innodbt where id = 2;

-- 可以对非加锁的行做任何操作
INSERT INTO `test`.`innodbt`(`id`,`name`)VALUES(4,'test');
update test.innodbt set name = 'ttt' where id = 4;
delete from test.innodbt where id = 4;

-- 可以对其他表做任何操作
select * from test.myisamt;
INSERT INTO `test`.`myisamt`(`id`,`name`)VALUES(8,'aaa');

rollback;
```

事务B

```sql
-- 另外一个事务加独占锁时
begin;
-- 可以查询
select * from test.innodbt;
-- 更新进入阻塞
update test.innodbt set name = 'popp' where id = 2;

-- 获取独占锁进入阻塞
select * from test.innodbt where id = 2 for update;
-- 获取共享锁进入阻塞
select * from test.innodbt where id = 2 lock in share mode;
rollback;
```

## 常见问题

问题：数据量很大，访问量也很大，白天跟晚上的访问量没什么区别，怎么修改表的字段(修改表结构会锁表)

这时候就可以通过复制表来完成这件事情，然后通过触发器实现数据的同步。先创建新表 A，修改好表结构将原表的数据copy到新表A中，并在原表添加触发器，同步操作到A表中，copy完成后将A表rename为原来的表名，默认会删除原表

当然这样的操作其实已经有工具帮我们实现，基于触发器的`pt-online-schema-change`。安装这个需要安装 perl 环境，还要安装 [percona-toolkit 工具集合](https://www.percona.com/downloads/percona-toolkit/LATEST/)。使用例子： `perl ..\pt-online-schema-change h=127.0.0.1,p=123456,u=root,D=test,t=innodbt --alter "modify name varchar(150) not null default '' " --execute` 其中 h 是 host，p是 password，u 是用户，D 是 schema 也可以说是数据库，t 是表名，--alter 是修改的语句

当然还有不是基于触发器的开源工具，[gh-host](https://github.com/github/gh-ost)这个是通过 binlog 进行操作的

今天就聊到这里吧，夜深了，就不聊了，怕吵到别人。