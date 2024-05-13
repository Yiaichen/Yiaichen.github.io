---
title: MYSQL锁以及死锁的产生跟解决
date: 2018-08-17 23:22:59
tags: [MYSQL]
categories: MYSQL
---

### InnoDB
>MySQL InnoDB存储引擎，实现的是基于多版本的并发控制协议——[MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control) (Multi-Version Concurrency Control) 

>在MVCC并发控制中，读操作可以分成两类：
快照读 (snapshot read)与当前读 (current read)。
快照读，读取的是记录的可见版本 (有可能是历史版本)，不用加锁。
当前读，读取的是记录的最新版本，并且，当前读返回的记录，都会加上锁，保证其他事务不会再并发修改这条记录。

> 快照读
```sql
select * from table where ?;
```

> 当前读
```sql
select * from table where ? lock in share mode;
select * from table where ? for update;
insert into table values (…);
update table set ? where ?;
delete from table where ?;
```

>所有以上的语句，都属于当前读，读取记录的最新版本。并且，读取之后，还需要保证其他并发事务不能修改当前记录，对读取记录加锁。
其中，除了第一条语句，对读取记录加S锁 (共享锁)外，其他的操作，都加的是X锁 (排它锁)。


#### InnoDB锁类型

##### X锁 or S锁
在InnoDb中实现了两个标准的行级锁，可以简单的看为两个读写锁:
>* S-共享锁：又叫读锁，其他事务可以继续加共享锁，但是不能继续加排他锁。
>* X-排他锁: 又叫写锁，一旦加了写锁之后，其他事务就不能加锁了。

兼容性：是指事务A获得一个某行某种锁之后，事务B同样的在这个行上尝试获取某种锁，如果能立即获取，则称锁兼容，反之叫冲突。

| . | X | S |
| :---------:| :---------: | :-------: |
| X | 冲突 | 冲突 |
| S | 冲突 | 兼容 |

##### 意向锁
意向锁在InnoDB中是表级锁,和他的名字一样他是用来表达一个事务想要获取什么。意向锁分为:
>* IS-意向共享锁(lock in share mode):表达一个事务想要获取一张表中某几行的共享锁。
>* IX-意向排他锁(for update):表达一个事务想要获取一张表中某几行的排他锁。

>这个锁有什么用呢？为什么需要这个锁呢？ 
首先说一下如果没有这个锁，如果要给这个表加上表锁，一般的做法是去遍历每一行看看他是否有行锁，这样的话效率太低。
而我们有意向锁，只需要判断是否有意向锁即可，不需要再去一行行的去扫描。

| . | IX | IS | X | S |
| :---------:| :---------: | :-------: | :-------: | :-------: |
| IX | 兼容 | 兼容 | 冲突 | 冲突 |
| IS | 兼容 | 兼容 | 冲突 | 兼容 |
| X | 冲突 | 冲突 | 冲突 | 冲突 |
| S | 冲突 | 兼容 | 冲突 | 兼容 |

**注：意向锁之间兼容，排它锁（包含意向）跟共享锁冲突，共享锁（包含意向）之间是兼容的**


##### 间隙锁(gap锁)
<img src="/img/lock/gap.png">

>间隙锁顾名思义锁间隙，不锁记录。锁间隙的意思就是锁定某一个范围，间隙锁又叫gap锁，其不会阻塞其他的gap锁，但是会阻塞插入间隙锁，这也是用来防止幻读的关键。

| where条件 | 定位条件 | 终止条件 | 加锁范围 |
| :---------:| :---------: | :-------: | :-------: |
| ID < X | 	infinum | X | (infinum,X] |
| ID <= X | infinum | X的下一条记录 | (infinum,X的下一条记录] |
| ID > X | X的下一条记录 | maxnum | (X,maxnum] |
| ID >= X | X | maxnum | [X,maxnum] | 



#### 事务隔离级别
```sql
SHOW VARIABLES LIKE '%isolation%';
```
>* Read Uncommited：可以读取未提交记录。此隔离级别，不会使用，忽略。
>* Read Committed (RC)：针对当前读，RC隔离级别保证对读取到的记录加锁 (记录锁)，存在幻读现象。
>* Repeatable Read (RR)：针对当前读，RR隔离级别保证对读取到的记录加锁 (记录锁)，同时保证对读取的范围加锁，新的满足查询条件的记录不能够插入 (间隙锁)，不存在幻读现象。
>* Serializable：Serializable隔离级别下，读写冲突，因此并发度急剧下降，在MySQL/InnoDB下不建议使用。

#### 查看当前锁的状态
```sql
SELECT * FROM information_schema.INNODB_LOCKS;
```

|Column|Description|
| :---------:| :---------: |
|lock_id	|锁ID|
|lock_trx_id|事务ID, 可以连INNODB_TRX表查事务详情|
|lock_mode	|锁的模式： S, X, IS, IX, S_GAP, X_GAP, IS_GAP, IX_GAP, or AUTO_INC|
|lock_type	|锁的类型: 行级锁 或者表级锁|
|lock_table	|加锁的表|
|lock_index	|如果是lock_type='RECORD' 行级锁 ,为锁住的索引，如果是表锁为null|
|lock_space	|如果是lock_type='RECORD' 行级锁 ,为锁住对象的Tablespace ID，如果是表锁为null|
|lock_page	|如果是lock_type='RECORD' 行级锁 ,为锁住页号，如果是表锁为null|
|lock_rec	|如果是lock_type='RECORD' 行级锁 ,为锁住页号，如果是表锁为null|
|lock_data	|事务锁住的主键值，若是表锁，则该值为null|


### 一句简单的加锁分析
```sql
SQL1：select * from t1 where id = 10;
SQL2：delete from t1 where id = 10;
```

#### 前提
>* 前提一：当前系统的隔离级别是什么？
>* 前提二：id列是不是主键？
>* 前提三：id列如果不是主键，那么id列上有索引吗？
>* 前提四：id列上如果有二级索引，那么这个索引是唯一索引吗？

#### 常用组合
>* 组合一：id列是主键，RC隔离级别
>* 组合二：id列是二级唯一索引，RC隔离级别
>* 组合三：id列是二级非唯一索引，RC隔离级别
>* 组合四：id列上没有索引，RC隔离级别
>* 组合五：id列是主键，RR隔离级别
>* 组合六：id列是二级唯一索引，RR隔离级别
>* 组合七：id列是二级非唯一索引，RR隔离级别
>* 组合八：id列上没有索引，RR隔离级别
>* 组合九：Serializable隔离级别

#### 分析常用组合
##### 组合一
<img src="/img/lock/medish.jpg">

> 主键为id,隔离级别是RC，只需要在主键上加`X锁`。

##### 组合二
<img src="/img/lock/medish2.jpg">

> id为唯一索引,假设主键为name,隔离级别是RC。由于id是unique索引，因此delete语句会选择走id列的索引进行where条件的过滤，
在找到id=10的记录后，首先会将unique索引上的id=10索引记录加上X锁，同时，会根据读取到的name列，回主键索引(聚簇索引)，
然后将聚簇索引上的name = 'd' 对应的主键索引项加`X锁`。

问题：为什么聚簇索引上的记录也要加锁？
> 如果并发的一个SQL，是通过主键索引来更新：`update t1 set id = 100 where name = 'd';` 
此时，如果delete语句没有将主键索引上的记录加锁，那么并发的update就会感知不到delete语句的存在，
违背了同一记录上的更新/删除需要串行执行的约束。

##### 组合三
<img src="/img/lock/medish3.jpg">

> id为非唯一索引,隔离级别是RC,相对于组合一、二,组合三又发生了变化,id列上的约束又降低了，id列不再唯一，只有一个普通的索引。
首先，id列索引上，满足id = 10查询条件的记录，均已加锁。同时，这些记录对应的主键索引上的记录也都加上了锁。
与组合二唯一的区别在于，组合二最多只有一个满足等值查询的记录，而组合三会将所有满足查询条件的记录都加锁。

##### 组合四
<img src="/img/lock/medish4.jpg">

> id无索引,隔离级别是RC,由于id列上没有索引，因此只能走聚簇索引，进行全部扫描。
从图中可以看到，满足删除条件的记录有两条，但是，聚簇索引上所有的记录，都被加上了X锁。
无论记录是否满足条件，全部被加上X锁。既不是加表锁，也不是在满足条件的记录上加行锁。

##### 组合五 = 组合一

##### 组合六 = 组合二

##### 组合七
<img src="/img/lock/medish5.jpg">

> id为非唯一索引,隔离级别是RR,首先，通过id索引定位到第一条满足查询条件的记录，加记录上的X锁，加GAP锁，然后加主键聚簇索引上的记录X锁，然后返回；
然后读取下一条，重复进行。直至进行到第一条不满足条件的记录，此时，不需要加记录X锁，但是仍旧需要加GAP锁，最后返回结束。

##### 组合八
<img src="/img/lock/medish6.jpg">

> 如图，这是一个很恐怖的现象。首先，聚簇索引上的所有记录，都被加上了X锁。其次，聚簇索引每条记录间的间隙(GAP)，也同时被加上了GAP锁。这个示例表，只有6条记录，一共需要6个记录锁，7个GAP锁。试想，如果表上有1000万条记录呢？
在这种情况下，这个表上，除了不加锁的快照度，其他任何加锁的并发SQL，均不能执行，不能更新，不能删除，不能插入，全表被锁死。
所以在Repeatable Read隔离级别下，如果进行全表扫描的当前读，那么会锁上表中的所有记录，同时会锁上聚簇索引内的所有GAP，杜绝所有的并发 更新/删除/插入 操作。

##### 组合九
> Serializable隔离级别，影响的是`SQL1：select * from t1 where id = 10;` 这条SQL。
在RC，RR隔离级别下，都是快照读，不加锁。但是在Serializable隔离级别，SQL1会加读锁，也就是说快照读不复存在。
在MySQL/InnoDB中，所谓的读不加锁，并不适用于所有的情况，而是隔离级别相关的。Serializable隔离级别，读不加锁就不再成立，所有的读操作，都是当前读。



### 举个栗子

#### 死锁
<img src="/img/lock/dead.png">

> 死锁:是指两个或两个以上的事务在执行过程中，因争夺资源而造成的一种互相等待的现象。
说明有等待才会有死锁，解决死锁可以通过去掉等待，比如回滚事务。
如果出现回滚，通常来说InnoDB会选择回滚权重较小的事务，也就是`undo`较小的事务。

> 假设我们有一个用户表,然后插入几条数据：

```sql
CREATE TABLE `user` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(11) CHARACTER SET utf8mb4 DEFAULT NULL,
  `comment` varchar(11) CHARACTER SET utf8 DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `index_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
```
```sql
insert user select 20,333,333;
insert user select 25,555,555;
insert user select 20,999,999;
```

> 然后我们开启事务 `A` 跟 事务`B`：

| 事件点 | 事务A | 事务B |
| :---------:| :---------: | :-------: |
| 1 | begin; |   |
| 2 |   | begin; |
| 3 | delete from user where name = '777'; |   |
| 4 |   | delete from user where name = '666'; |
| 5 |   | insert user select 26,'666','666'; |
| 6 | insert user select 27,'777','777'; |   |
| 7 | ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction |   |
| 8 |   | Query OK, 1 row affected (14.32 sec) Records: 1 Duplicates: 0 Warnings: 0 |

**可以看见事务A出现被回滚了，而事务B成功执行。 具体每个时间点发生了什么呢?**
>* 时间点3：事务A删除需要对777这个索引加上X锁，但是其不存在，所以只对555-999之间加间隙锁
>* 时间点4：同理事务B也对555-999之间加间隙锁。间隙锁之间是兼容的,所以正常执行，此时两边都有一个间隙锁
>* 时间点5：事务B，执行Insert操作，首先插入意向锁，但是555-999之间有间隙锁，由于插入意向锁和间隙锁冲突，事务B阻塞，等待事务A释放间隙锁
>* 时间点6：事务A同理，等待事务B释放间隙锁。于是出现了A->B,B->A回路等待。
>* 时间点7：事务管理器检查到死锁选择回滚事务A
>* 时间点8：事务B插入操作执行成功。

<img src="/img/lock/deadlock.png">


#### 解决方案
>* 方案一：隔离级别降级为RC，在RC级别下不会加入间隙锁，所以就不会出现毛病了，但是在RC级别下会出现幻读，可提交读都破坏隔离性的毛病，所以这个方案不行。
>* 方案二：较少的修改代码逻辑，在删除之前，可以通过快照查询(不加锁)，如果查询没有结果，则直接插入，如果有通过主键进行删除，在之前第三节实验2中，通过唯一索引会降级为记录锁，所以不存在间隙锁。

#### 防止死锁
>* 以固定的顺序访问表和行。交叉访问更容易造成事务等待回路。
>* 尽量避免大事务，占有的资源锁越多，越容易出现死锁。建议拆成小事务。
>* 降低隔离级别。如果业务允许，将隔离级别调低也是较好的选择，比如将隔离级别从RR调整为RC，可以避免掉很多因为gap锁造成的死锁。
>* 为表添加合理的索引。防止没有索引出现表锁，出现的死锁的概率会突增。


### 总结
做一个简单的总结，要做的完全掌握MySQL/InnoDB的加锁规则，甚至是其他任何数据库的加锁规则，需要具备以下的一些知识点：

>* 了解数据库的一些基本理论知识：数据的存储格式 (堆组织表 vs 聚簇索引表)；并发控制协议 (MVCC vs Lock-Based CC)；Two-Phase Locking；数据库的隔离级别定义 (Isolation Level)；
>* 了解SQL本身的执行计划 (主键扫描 vs 唯一键扫描 vs 范围扫描 vs 全表扫描)；
>* 了解数据库本身的一些实现细节 (过滤条件提取；Index Condition Pushdown；Semi-Consistent Read)；
>* 了解死锁产生的原因及分析的方法 (加锁顺序不一致；分析每个SQL的加锁顺序)






















