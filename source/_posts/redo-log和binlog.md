---
title: redo log和binlog
date: 2020-07-26 10:07:09
tags:
	- others
	- DB
---



MySql可以恢复到半个月内任意一秒的状态

<!--more-->

## 更新数据库字段

创建一个表：

```sql
CREATE TABLE T(ID int primary key, c int) 
```

如果要将ID=2这一行的值加1，SQL语句就会这么写：

```sql
UPDATE T set c= c+1 where ID=2
```

更新语句执行步骤：

- 连接器连接数据库
- 分析器分析到这是一条更新语句
- 优化器决定使用ID
- 执行器负责找到这一行，操作引擎进行更新

## 涉及日志

不同点：

更新流程还设计到两个重要的日志模块：**redo log（重做日志）和binlog（归档日志）**

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595641298348-b119c308-bdda-48d1-bed4-69dd7d0cec36.png)

ib_logfile$

就是redo log

### redo log

《孔乙己》酒店赊账：

- 客人不多，写在黑板上
- 赊账客人多，把黑板上的内容记录在账本上



如果要赊账

1. 直接把账本翻出来，更新账本
2. 写在黑板上，打烊之后更新账本



如果柜台很忙，老板一定会选择第二种，因为相对于黑板，账本肯定是内容多而查找时间长的。

MySql更新也是同样的道理：

如果每一次更新都要写进磁盘，那么磁盘也要找到对应的那条记录，然后再更新



#### WAL(WRITE-Ahead Logging)技术

先写日志，再写磁盘



当有一条记录需要更新时，InnoDB引擎会先把记录写到 redo log里，更新内存

> P.s redo log 是不是这个日志需要再redo，重新执行的意思

InnoDB会在系统比较空闲的时候更新磁盘。



redo log更新过程：

- 如果今天赊账不多，老板会在打烊之后更新账本
- 如果今天赊账多，黑板(redo log)写满了，老板要放下手头的活，更新账本，擦掉黑板



同样的，InnoDB的redo log是固定大小的，分配一组为m个文件，每个文件大小为nGB

黑板的总大小：mn GB

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595642539184-2fef52ca-a128-4572-abb9-42af91b5e890.png)

这个记录过程是循环单链表式的。

checkpoint跟write pos 是相同方向推进的。

如果 write pos 追上了checkpoint，表示“黑板”满了，这时候不能执行新的更新，要停下来先把redo log的内容写到磁盘中，把check point 推进一下。



#### crash-safe

redo log 保证数据库发生异常重启，之前提交的记录也不会丢失，这个能力称为 crash-safe



### binlog：日志模块

MySql 整体来看分为两块：

- Server层
- 引擎层

redo log 是InnoDB引擎特有的日志，而Server层也有自己的日志，称为binlog(归档日志)



为什么会有两份日志？

原因1：一开始MySql里没有InnoDB引擎

原因2：即使用MySql，用户也不一定用的是InnoDB引擎

原因3：MySql自带引擎是MylSAM，没有crash-safe能力，binlog只用于归档，所以InnoDB设计了redo log



两个日志的不同点(都不是同一个层面的东西为什么要放在一起比较啊(′д｀ )…彡…彡)

- redo log 是InnoDB引擎特有，binlog所有引擎都可以用
- redo log 是物理日志，记录的是“在某个数据页上做了什么修改”（记录了这个页做了什么改动）,binlog是逻辑日志，记录的是这个语句的原始逻辑，比如，“给ID=2这一行的c字段加1”。（binlog有两种模式，statement模式记sql语句，row格式记录行的内容，记两条，更新前和更新后都有。）
- redo log 循环写，binlog 是追加写，不会覆盖。



again：执行器和InnoDB引擎在执行update语句时的内部流程



![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595645940148-05786141-11c7-4947-a388-ea6b24623794.png)

解析：

step1：执行器取到id =2 这一行（id是主键，引擎很快能查到这一行）

如果在内存中，直接把这一行返回给执行器，如果不在内存中，从磁盘中取出来返回给执行器

step2：执行器 对这行记录执行sql操作

step3：执行完，引擎把新行写入内存

step4：日志操作。事务的两段提交。





> P.s 这个流程图还没有涉及到redo log 何时执行 checkpoint 这个操作。



### 两阶段提交



binlog记录所有的逻辑操作，如果半个月内的数据库都可以恢复，那么备份系统中一定保存最近半个月的所有binlog。



如果系统定期做了整库备份：

- 找到最近一次的全量备份，恢复到临时库
- 从备份的时间点开始，把备份的binlog依次取出来，重放到误删表之前的时刻



![img](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595659924346-9ec62427-ebb8-4058-9b38-dbb40a2218b7.png)

两阶段提交的过程：

简易版本：



![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595660803750-4818acef-fc2e-4b06-9b6e-3dfe409760be.png)

![img](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595662416707-9a1f2e74-e861-4a8d-a301-7c5c7ef463ae.png)

详细解释版本：

prepare：redolog写入log buffer，并fsync持久化到磁盘，在redolog事务中记录2PC（两阶段提交）的XID，在redolog事务打上prepare标识

commit：binlog写入log buffer，并fsync持久化到磁盘，在binlog事务中记录2PC的XID，同时在redolog事务打上commit标识

其中，prepare和commit阶段所提到的“事务”，都是指内部XA事务，即2PC





为什么日志需要“两阶段提交”？

redo log和binlog是两个独立的逻辑，而事务提交必须确保两者`同时有效`。不然会出现不一致的情形。

如果不用两阶段提交(也就是redo log 的prepare 跟commit 是一起的)，那么提交顺序有两种，如果在写完第一个日志后crash，第二个日志还没有写完，会出现的情况：

（1）先redo log后binlog

redo log 已经写完， 系统crash。

引擎通过redo log恢复数据库，c=1

如果通过binlog 恢复数据库，c=0

（2）先binlog 后redo log

binlog已经写完，系统crash

引擎通过redo log恢复数据库，c=0

如果通过bin log 恢复，c=1



由于要保证2份日志的一致，所以可以把日志提交看作一个事务，不能让中间环节出现，也就是不能一个成功一个失败。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595663300063-65417753-c9e4-41f1-a61f-62c1dbdfe1ce.png)

第一种情况：

A处宕机：

redo log 处于prepare，binlog未提交

引擎回滚事务跟binlog恢复的结果相同

第二种情况：

B处宕机：

redolog处于prepare但是binlog完整，提交事务。

跟binlog恢复出来的数据库一致。



引出崩溃恢复时的逻辑：



1. 如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；
2. 如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：
   a. 如果是，则提交事务；
   b. 否则，回滚事务。



丁奇：

[MySQL 中 6 个常见的日志问题](https://www.infoq.cn/article/M6g1yjZqK6HiTIl_9bex)

> awsl

### 问题 1：MySQL 怎么知道 binlog 是完整的?

回答：一个事务的 binlog 是有完整格式的：

- statement 格式的 binlog，最后会有 COMMIT；
- row 格式的 binlog，最后会有一个 XID event。

另外，在 MySQL 5.6.2 版本以后，还引入了 binlog-checksum 参数，用来验证 binlog 内容的正确性。对于 binlog 日志由于磁盘原因，可能会在日志中间出错的情况，MySQL 可以通过校验 checksum 的结果来发现。所以，MySQL 还是有办法验证事务 binlog 的完整性的。

### 问题 2：redo log 和 binlog 是怎么关联起来的?

回答：它们有一个共同的数据字段，叫 XID。崩溃恢复的时候，会按顺序扫描 redo log：

- 如果碰到既有 prepare、又有 commit 的 redo log，就直接提交；
- 如果碰到只有 parepare、而没有 commit 的 redo log，就拿着 XID 去 binlog 找对应的事务。

### 问题 3：处于 prepare 阶段的 redo log 加上完整 binlog，重启就能恢复，MySQL 为什么要这么设计?

回答：其实，这个问题还是跟我们在反证法中说到的数据与备份的一致性有关。在时刻 B，也就是 binlog 写完以后 MySQL 发生崩溃，这时候 binlog 已经写入了，之后就会被从库（或者用这个 binlog 恢复出来的库）使用。

所以，在主库上也要提交这个事务。采用这个策略，主库和备库的数据就保证了一致性。

### 问题 4：如果这样的话，为什么还要两阶段提交呢？干脆先 redo log 写完，再写 binlog。崩溃恢复的时候，必须得两个日志都完整才可以。是不是一样的逻辑？

回答：其实，两阶段提交是经典的分布式系统问题，并不是 MySQL 独有的。

如果必须要举一个场景，来说明这么做的必要性的话，那就是事务的持久性问题。

对于 InnoDB 引擎来说，如果 redo log 提交完成了，事务就不能回滚（如果这还允许回滚，就可能覆盖掉别的事务的更新）。而如果 **redo log 直接提交，然后 binlog 写入的时候失败，InnoDB 又回滚不了，数据和 binlog 日志又不一致了。**

两阶段提交就是为了给所有人一个机会，当每个人都说“我 ok”的时候，再一起提交。



### 问题5：只用redo log可以吗？

只从崩溃恢复的角度来讲是可以的。你可以把 binlog 关掉，这样就没有两阶段提交了，但系统依然是 crash-safe 的。



但是binlog有其他功能：

- 归档
- MySQL 系统依赖于 binlog。binlog 作为 MySQL 一开始就有的功能，被用在了很多地方。其中，MySQL 系统高可用的基础，就是 binlog 复制。
- 很多公司有异构系统（比如一些数据分析系统），这些系统就靠消费 MySQL 的 binlog 来更新自己的数据。关掉 binlog 的话，这些下游系统就没法输入了。

### 问题6：只用binlog可以吗？

现阶段的binlog做不到崩溃恢复。



![img](https://cdn.nlark.com/yuque/0/2020/jpeg/1762736/1595664361475-39801db7-d5bc-4d9e-ae1d-f30a1b375797.jpeg)



如果做成这样，在binlog2未commit之前crash，



重启后，引擎内部事务 2 会回滚，然后应用 binlog2 可以补回来；但是对于事务 1 来说，系统已经认为提交完成了，不会再应用一次 binlog1。



（因为MySql的Server层不支持WAL），所以不会再应用一次binlog1.





## 如何通过bin log 恢复数据库

1. 查看bin log是否开启

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595647597227-11b21c4f-18fb-4f51-93b6-332eddb67ee2.png)



2. 更多log内容

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1595648035540-f62ff990-5b52-41e1-bb25-e682707d3a6a.png)



3. 

































本节问题：

- 🔲1. Mysql哪些内容存在内存中？哪些存在磁盘中?
- 🔲2. 定期全量备份的周期“取决于系统重要性，有的是一周一备，有的是一天一备”

答案：取决于系统对恢复时间的要求。如果要求系统尽快恢复，那么就要用一天一备，如果对系统恢复时间没有要求，那么可以用一周一备。因为一周一备就要用到一周前的全量备份+这一周的binlog。一天一备是用一天前的全量备份＋这一天的binlog，显然后者恢复起来更快。