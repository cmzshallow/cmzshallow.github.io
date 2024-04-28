---
title: MySQL事务
date: 2024-04-29 10:03
tags:
  - MySQL
categories:
  - 学习
---

## 个人理解，不对的地方请指正

当前系统 MySQL 版本为：8.0.28
### 事务的隔离级别：
 - 读未提交（Read Uncommitted）：另一个事务未提交的修改结果，本事务也可以看到。
 - 读已提交（Read committed）：另一个事务将修改结果提交后，本事务才可以看到。（Oracle 数据库默认事务级别）
 - 可重复读（Repetable Read）：一个事务的执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。（MySQL 数据库默认事务级别）
 - 串行化（Serializable）：对于同一行数据，读会加读锁，写会加写锁，当读写锁冲突时，后访问的事务必须等前一个事务执行完成后才能访问。

### 查看当前数据库连接的事务的隔离级别：
 - ```show variables like '%transaction_isolation%';```
 - ```select @@transaction_isolation;```
 - ```SELECT @@session.transaction_isolation;```

### 查看全局事务的隔离级别：
 - ```SELECT @@global.tx_isolation;```

### 修改事务的隔离级别：
 - ```set [session | global] transcation isolation level {read uncommitted | read committed | repeatable read | serializable}```
 - session：表示修改的事务隔离级别应用于当前数据库连接内的所有事务
 - global：表示修改的事务隔离级别应用于所有数据库连接中的所有事务，但是已经创建的数据库连接不受影响
 - 如果省略 session 和 global，表示修改的事务隔离级别将应用于当前 session 内的下一个还未开始的事务
 - 任何用户都有权限修改自己的数据库连接的事务隔离级别，但是只有拥有 super 权限的用户才能修改全局的事务隔离级别

### 事务隔离级别的实现：
 - 数据库会创建一个一致性读视图（跟 create view 创建的视图不是同一个视图），访问的时候以视图的逻辑结果为准。
 - 读未提交：直接返回记录的最新值，没有视图概念
 - 读已提交：每个 SQL 语句开始执行时创建一个视图
 - 可重复读：事务启动时创建，整个事务期间都使用这个视图
 - 串行化：通过加锁避免并行访问，不需要创建视图

### 需要注意的是：
```begin/start transaction;``` 命令并不是一个事务的起点，在执行到它们之后的第一个操作 InnoDB 表的语句，事务才真正启动。如果想要马上启动一个事务，可以使用 ```start transaction with consistent snapshot;``` 这个命令。
第一种启动方式，一致性视图是在执行第一个快照读语句时创建的；
第二种启动方式，一致性视图是在执行 ```start transaction with consistent snapshot``` 时创建的。

### 视图的数据来源：
 - InnoDB 里面每个事务有一个唯一的事务 ID，叫作transaction id。它是在事务开始的时候向 InnoDB 的事务系统申请的，按照申请顺序严格递增。
 - 每条数据在更新的时候，同时也会记录一条回滚操作到回滚日志（undo log）中，并且记录对应的 row_trx_id（事务ID），通过回滚操作，就可以根据 row_trx_id 得到对应版本的数据。
 - InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前所有启动了，但是未提交的事务的 ID。数组里面事务 ID 的最小值记为低水位，当前系统里面已经创建过的事务 ID 的最大值加 1 记为高水位。
 - 这个数组和高低水位，就组成了当前事务的一致性视图（read-view）。
 
具体流程如下：
假设数据库中有表 t，结构如下：
```sql
create table t (
    `id` int not null primary key auto_increment,
    `num` int not null
) engine = innodb;
```
并且假设事务1插入了两条数据：
```sql
insert into t (`id`, `num`) values (1, 1), (2, 2);
```
那么此时数据库中的数据为：

![image.png](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/d4cc7898afe2d875bea10fb81ff1f0b3.png)

假设此时有三个事务同时进行操作，操作内容如下：
| 时间 | A | B | C |
| :--: | :--: | :--: | :--: |
|  T1  | begin;<br/>update t set num = 3 where id = 1 | begin;<br/>update t set num = 4 where id = 2;<br/>commit; |                               |
|  T2  |                                               |                                                             | begin;<br />select * from t;  |
|  T3  |                    commit;                    |                                                             |                               |
|  T4  |                                               |                                                             | select * from t;<br/>commit; |

假设事务 A 的事务 ID 为 2，事务 B 的事务 ID 为 3，事务 C 的事务 ID 为 4，当 T1 时刻结束后，回滚日志中有：
|  id  | num  | row_trx_id |
| :--: | :--: | :--------: |
|  1   |  3   |     2      |
|  1   |  1   |     1      |

|  id  | num  | row_trx_id |
| :--: | :--: | :--------: |
|  2   |  4   |     3      |
|  2   |  2   |     1      |

当 T2 时刻事务 C 启动事务后，根据 InnoDB 给事务创建一致性视图的规则，当前活跃的事务 ID 为 2，那么低水位就是 2。高水位则为系统里面创建过的事务 ID 的最大值加 1，即为事务 C 的事务 ID + 1，也就是 5。由于事务 B 已提交，所以活跃事务数组中只有事务 A 的事务 ID，也就是 2。那么此时可以得到下图：

![Screenshot_20230919_181334.jpg](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/b9b1aa2aa2c6f1e56c1b59f674097183.jpg)

对于当前事务的启动瞬间来说，一个数据版本的 row_trx_id，有以下几种可能：
 - 如果 row_trx_id < 2，表示这个版本是已提交的事务生成的，这个数据是可见的；
 - 如果 row_trx_id >= 5，表示这个版本是由将来启动的事务生成的，是肯定不可见的；
 - 如果 2<= row_trx_id < 5，那就包括两种情况
   - 若 row trx_id 在数组中，表示这个版本是由还没提交的事务生成的，不可见；
   - 若 row trx_id 不在数组中，表示这个版本是已经提交了的事务生成的，可见。

所以，由于可重复读的事务隔离级别只会在事务开始时创建一次一致性视图，所以即时后面事务 A 将事务提交了，事务 C 的一致性视图也不会变更，事务 A 操作的数据对于事务 C 来说依然还是不可见的。

但如果是读提交的事务隔离级别，当 T3 时刻事务 A 提交事务后，T4 时刻事务 C 查询数据时，一致性视图就变成了：

![Screenshot_20230919_181334.jpg](https://shallowcmz.oss-cn-hangzhou.aliyuncs.com/aurora/articles/6dcecb3d650ed9858f4ca8caee219980.jpg)

由于此时低水位和活跃数组都已经改变，所以事务 A 的提交的修改结果对于事务 C 来说就是可见的。

上述就是我理解的 InnoDB MVCC 的实现。

当然，这只是针对普通的查询，如果是修改语句，或者修改一下上述的查询语句，改为：

```update t set num = 5 where id = 2```

或：

```select * from t lock in share mode```

或：

```select * from t for update```

那么也可以获取到最新版本的数据，这里用到了一条规则：更新数据都是先读后写的，而这个读，只能读当前的值，称为当前读（current read）。因此，update 的时候，会获取当前的 num 值，并更新为 5，然后在回滚日志中记下（2， 5， 4）这一条记录，此时这条记录对应的事务 id 为事务 C 的事务 id：5，当然就对事务 C 可见了。

此外，虽然当前读针对的是 update 语句，但是 select 语句加锁的话也是当前读。其中 ```lock in share mode``` 加了读锁（S锁，共享锁），```for update``` 加了写锁（X锁，排他锁）。

当然，由于以上操作都对数据进行加锁操作，所以如果别的线程想要修改数据，则会被事务 C 查询/修改过程中加的锁给拦住，需要等事务 C 将对应的锁释放后才可以进行操作，具体哪些数据不能改则要看事务 C 扫描了哪些数据，上了什么锁了。
