---
title: 【系统学习-mysql】【三】事务
categories:
  - mysql 学习
tags:
  - mysql
date: 2019-10-18 11:13:33
---

## 概要

今天来浅聊一下 mysql 的事务，我们在业务中，需要确保业务的逻辑正确，比如账户A转给账户B，应该是A的余额减少，B的余额增加，或者用户下一个单，需要创建好几条数据等，这时候一般需要几条 sql 语句去执行，如果只执行了其中一部分 sql，那么业务逻辑就会出现错误，比如只执行了账户A余额的扣减，或者用户下单只创建了一部分数据。需要解决这类问题，我们就可以使用事务。但是当有多个事务需要读取和操作同一条数据的时候，事务之间还需要一定的规则去约定，这时候就有不同级别的隔离性规则。所以我们今天聊一下以下几点

- 开启事务的语法
- 事务的特性 ACID
- 事务的隔离性

<!-- more -->

## 开启事务的语法

mysql 开启事务很简单，执行 begin 就完事了。不过要注意，mysql 中**支持事务的存储引擎只有 innodb**，可以通过执行` show engines`查看存储引擎是否支持事务

```sql
begin;
update table xxxx;
-- 提交
commit;
-- 回滚
rollback;
```

其实还有一个叫还原点(savepoint)的东西，只是我们平时没有使用到，savepoint 可以回滚到具体的保存点上，我们来看一下例子。当执行 `rollback to savepoint s1`后，事务会回滚到`s1`，即后面两条 update 语句被回滚了

```sql
-- 先开启事务
begin;
update test.innodbt set name = 'savepoint1' where id = 2;
savepoint s1;
update test.innodbt set name = 'savepoint2' where id = 3;
savepoint s2;
update test.innodbt set name = 'savepoint3' where id = 4;
-- 回滚到具体的 savepoint
rollback to savepoint s1;

commit;
```

## 事务的特性 ACID

事务性数据库是可以为一系列数据库操作(begin到commit)提供ACID属性的数据库操作系统。ACID 就是值事务的特性，分别为

- **原子性(Atomicity)**。要么全执行，要么全不执行
- **一致性(consistency)**。事务对数据的影响必须是符合数据库规则的(约束、触发器等)，不符合规则(比如语句报错)的操作不能影响数据
- **隔离性(isolation)**。事务之间相互不影响，未完成的事务操作结果在其他事务中不可见(除了 read uncommitted)
- **持久性(durability)**。事务一旦提交，就永久有效，就算数据库崩溃也不能影响该事务的提交结果。通常是指修改记录不在不稳定的内存中

## 事务的隔离性

隔离性决定着在并发情况下，一个事务在操作过程中，别的事务操作对该事务的可见程度。在低隔离级别下，事务可以立即查看到其他事务中未提交的操作；在高隔离级别下，事务可能通过加锁，阻塞住将要操作同样数据的事务，防止别的事务对该事务造成影响。可能这聊的比较抽象，接下来我们一起操作一下就清楚了。在操作之前，我们先来看一下隔离级别有几种

**隔离级别依次提高**

- **read uncommitted（读未提交）**
- **read committed（读已提交）**
- **repeatable read（可重复读）**
- **serializable（可串行化）**

mysql5.7 默认的隔离级别是 `repeatable read`，可以通过执行 `show variables like '%tx_isolation%'` 查看当前事务的隔离级别

### 读未提交 & 脏读

read uncommitted，读未提交是指可以读取到其他事务未提交的结果

我们接下来会使用到的数据

| id | balance |
| :--: | :--: |
| 1 | 300 |
| 2 | 200 |
| 3 | 100 |

事务A

```sql
-- 将隔离级别设置为读未提交
set session transaction isolation level read uncommitted;

begin;
update test.account set balance = balance - 50 where id = 1;
rollback;
```

事务B

```sql
set session transaction isolation level read uncommitted;
begin;
-- 在事务A还未提交的时候，事务B已经可以读取到事务A的修改，即 balance 为 250
select * from test.account where id = 1;
```

脏读是指，事务A做了修改操作但没提交或回滚，事务B读取到了 balance 为 250，然后事务A回滚了，这时候事务B拿到的 250 这个数据就是脏数据，如果事务B接下拿这个数据进行操作，那么就会出问题

### 读已提交 & 不可重复读

read committed，读已提交是指只能读取到事务提交后的结果，由于不会读取到未提交的数据，所以也就没有脏读的问题，但也会产生不可重复读的问题。接下来我们继续看

事务A

```sql
set session transaction isolation level read committed;

begin;
update test.account set balance = balance - 50 where id = 1;
commit;
```

事务B

```sql
set session transaction isolation level read committed;
begin;
-- 当事务A更新了数据但未提交的时候，事务A的 balance 是 250，事务B这里 balance 是 300
-- 当事务A提交了时，事务B再读取就是 balance 为 250
select * from test.account where id = 1;
```

不可重复读是指，事务A做了修改操作但没有提交时，事务B读取到了 balance 为 300，然后事务A提交了，这时候事务B再读取得到了 balance 为 250，同一事务两次读取结果不一样，产生了诡异的现象

解决这个问题可以通过加锁来完成，即在事务B中将查询语句修改为`select * from test.account where id = 1 for update`。这个时候如果事务A已经进行了数据操作，那么事务B获取这个锁就会阻塞，知道事务A完成了工作；如果这个时候事务A还未进行数据操作，那么事务B直接获取到这个锁，但事务A想再操作这条数据的就需要等到这把锁释放掉。就这样，问题解决了

### 可重复读 & 幻读

repeatable read，可重复读是指事务开启后，第一次读取到的数据，再次读取不会因为其他事务做修改并提交而发生改变，保证了每次读取同一条数据都是一样的，就不会有不可重复读的问题。但也有幻读的问题，我们接下去继续看看

事务A

```sql
set session transaction isolation level repeatable read;

begin;
update test.account set balance = balance - 50 where id = 1;
commit;
```

事务B

```sql
set session transaction isolation level repeatable read;

begin;
-- 当事务A更新了数据但未提交，事务B读取到的 balance 是 300，这个跟 read committed 一样。但是事务A提交了，事务B再读取还是 300。也就是说不管读多少次，值都是一样的
-- 当事务A更新了数据并已提交，事务B读取到的 balance 是 250
select * from test.account where id = 1;
```

可以看出，可重复读就是不管其他事务是否有提交，只要当前事务读取过的数据，在事务内，再次读取就还是同样的数据。我们再来看一下幻读是怎么产生的

事务A

```sql
begin;
select * from test.account where id = 4;
INSERT INTO `test`.`account`(`id`,`balance`)VALUES(4,0);
commit;
```

事务B

```sql
begin;
select * from test.account where id = 4;
-- 在事务A没提交前，事务B读取到没有 id 为 4 的数据，由于可重复读，这时候，就算事务A提交了，读取到的结果也是一样的
-- 这时候信息满满地将数据插进去，发现主键冲突了
INSERT INTO `test`.`account`(`id`,`balance`)VALUES(4,0);
commit;
```

可以看到，由于可重复读，在操作过程中数据，明明没有主键冲突，但执行结果让人意外，就像幻觉一样？同样地，如果一个事务在批量操作，另外一个事务在插入数据，也会出现这样的现象。解决这个问题，就需要锁表了

特别注意，**在 repeatable read 隔离级别下，行锁的条件不是索引，将会锁整张表**。比如 `select * from test.account where balance = 200 for update`

### 可串行化

serializable，可串行化是指事务需要一个接着一个进行，不能并行地执行。比如事务A修改了数据a但未提交，事务B就不能修改甚至不能读取，进入阻塞；事务A读取了数据a但未提交，事务B能读取，但不能修改，进入阻塞

事务A

```sql
set session transaction isolation level serializable;

begin;
select * from test.account where id = 1;
update test.account set balance = balance - 50 where id = 1;
```

事务B

```sql
set session transaction isolation level serializable;

begin;
-- 当事务A执行了 update 操作，这里 select 进入阻塞
select * from test.account where id = 1;
-- 当事务A执行了 select 操作，这里 update 进入阻塞
update test.account set balance = balance - 50 where id = 1;
```

可以看到，串行化隔离级别会对读取或者修改的数据进行加锁，特别地，**如果读取的数据是整张表的数据，那么就会锁表**

那么我们来总结一下，其实可以看到，事务的隔离级别越高，越能保证数据的完整性和一致性，但并发性能就越低，对于多数应用程序而言，`read committted`就可以了。我们再来总结一下脏读、不可重复读和幻读

- 脏读：事务A读取了事务B更新的数据，然后B回滚了操作，那么事务A读取到的就是脏数据
- 不可重复读：事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据做了更新并提交，导致事务A多次读取同一数据时，结果不一致。解决这个问题可以锁行
- 幻读：事务A在事务B提交事务前，读取了数据，然后事务B先提交了，事务A在做提交，结果发现跟事务A想象的结果不一样。解决这个问题可以锁表

我们还可以再对脏读、不可重复读、幻读做一张表

| 事务隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :--: | :--: | :--: | :--: |
| 未提交读(read uncommitted) | 是 | 是 | 是 |
| 已提交读(read committed) | 否 | 是 | 是 |
| 可重复读(repeatable read) | 否 | 否 | 是 |
| 可串行化(serializable) | 否 | 否 | 否 |

Emmm，天气转凉，早点吃饭睡觉