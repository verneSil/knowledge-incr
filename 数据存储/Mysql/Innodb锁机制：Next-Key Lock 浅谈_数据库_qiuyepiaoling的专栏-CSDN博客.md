Innodb锁机制：Next-Key Lock 浅谈_数据库_qiuyepiaoling的专栏-CSDN博客

原文：http://www.cnblogs.com/zhoujinyi/p/3435982.html

数据库使用锁是为了支持更好的并发，提供数据的完整性和一致性。InnoDB是一个支持行锁的存储引擎，锁的类型有：共享锁（S）、排他锁（X）、意向共享（IX）、意向排他（IX）。为了提供更好的并发，InnoDB提供了非锁定读：不需要等待访问行上的锁释放，读取行的一个快照。该方法是通过InnoDB的一个特性：MVCC来实现的。

InnoDB有三种行锁的算法：

1. Record Lock：单个行记录上的锁。

2. Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。

3. nextt-Key Lock：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。


## 测试一：事务隔离级别：RR

```
root@localhost : test **10**:**56**:**10**>create table t(a int,key idx_a(a))engine =innodb;
Query OK, **0** rows affected (**0.20** sec)

root@localhost : test **10**:**56**:**13**>insert into t values(**1**),(**3**),(**5**),(**8**),(**11**);
Query OK, **5** rows affected (**0.00** sec)
Records: **5**  Duplicates: **0**  Warnings: **0** root@localhost : test **10**:**56**:**15**>select * from t; +------+
| a    |
+------+
|    **1** |
|    **3** |
|    **5** |
|    **8** |
|   **11** |
+------+
**5** rows in set (**0.00** sec)

section A:

root@localhost : test **10**:**56**:**27**>start transaction;
Query OK, **0** rows affected (**0.00** sec)

root@localhost : test **10**:**56**:**29**>select * from t where a = **8** for update; +------+
| a    |
+------+
|    **8** |
+------+
**1** row in set (**0.00** sec)

section B:
root@localhost : test **10**:**54**:**50**>begin;
Query OK, **0** rows affected (**0.00** sec)

root@localhost : test **10**:**56**:**51**>select * from t; +------+
| a    |
+------+
|    **1** |
|    **3** |
|    **5** |
|    **8** |
|   **11** |
+------+
**5** rows in set (**0.00** sec)

root@localhost : test **10**:**56**:**54**>insert into t values(**2**);
Query OK, **1** row affected (**0.00** sec)

root@localhost : test **10**:**57**:**01**>insert into t values(**4**);
Query OK, **1** row affected (**0.00** sec) ++++++++++ root@localhost : test **10**:**57**:**04**>insert into t values(**6**);

root@localhost : test **10**:**57**:**11**>insert into t values(**7**);

root@localhost : test **10**:**57**:**15**>insert into t values(**9**);

root@localhost : test **10**:**57**:**33**>insert into t values(**10**); 
++++++++++ 上面全被锁住，阻塞住了

root@localhost : test **10**:**57**:**39**>insert into t values(**12**);
Query OK, **1** row affected (**0.00** sec)

```

#### 问题：

为什么section B上面的插入语句会出现锁等待的情况？InnoDB是行锁，在section A里面锁住了a=8的行，其他应该不受影响。why？

分析：

因为InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围。上面索引有1，3，5，8，11，被Next-Key Locking的区间为：

（-∞,1），（1,3\]，（3,5\]，（5,8\]，（8,11\]，（11,+∞）

特别需要注意的是，InnoDB存储引擎还会对辅助索引下一个键值加上gap lock。如上面分析，那就可以解释了。

``` s
root@localhost : test **10**:**56**:**29**>select * from t where a = **8** for update; 
+------+
| a    |
+------+
| **8** |
+------+
**1** row in set (**0.00** sec)

```

该SQL语句锁定的范围是（5,8\]，下个下个键值范围是（8,11\]，所以插入5~11之间的值的时候都会被锁定，要求等待。即：插入5，6，7，8，9，10 会被锁住。插入非这个范围内的值都正常。

继续：插入超时失败后，会怎么样？

超时时间的参数：innodb\_lock\_wait_timeout ，默认是50秒。  
超时是否回滚参数：innodb\_rollback\_on_timeout 默认是OFF。

```

section A:

root@localhost : test **04**:**48**:**51**>start transaction;
Query OK, **0** rows affected (**0.00** sec)

root@localhost : test **04**:**48**:**53**>select * from t where a = **8** for update; +------+
| a    |
+------+
|    **8** |
+------+
**1** row in set (**0.01** sec)

section B:

root@localhost : test **04**:**49**:**04**>start transaction;
Query OK, **0** rows affected (**0.00** sec)

root@localhost : test **04**:**49**:**07**>insert into t values(**12**);
Query OK, **1** row affected (**0.00** sec)

root@localhost : test **04**:**49**:**13**>insert into t values(**10**);
ERROR **1205** (HY000): Lock wait timeout exceeded; try restarting transaction root@localhost : test **04**:**50**:**06**>select * from t; +------+
| a    |
+------+
|    **1** |
|    **3** |
|    **5** |
|    **8** |
|   **11** |
|   **12** |
+------+
**6** rows in set (**0.00** sec)

```

经过测试，不会回滚超时引发的异常，当参数innodb\_rollback\_on_timeout 设置成ON时，则可以回滚，会把插进去的12回滚掉。

默认情况下，InnoDB存储引擎不会回滚超时引发的异常，除死锁外。

  
既然InnoDB有三种算法，那Record Lock什么时候用？还是用上面的列子，把辅助索引改成唯一属性的索引。

## 测试二：

```

root@localhost : test **04**:**58**:**49**>create table t(a int primary key)engine =innodb;
Query OK, **0** rows affected (**0.19** sec)

root@localhost : test **04**:**59**:**02**>insert into t values(**1**),(**3**),(**5**),(**8**),(**11**);
Query OK, **5** rows affected (**0.00** sec)
Records: **5**  Duplicates: **0**  Warnings: **0** root@localhost : test **04**:**59**:**10**>select * from t; +----+
| a  |
+----+
|  **1** |
|  **3** |
|  **5** |
|  **8** |
| **11** |
+----+
**5** rows in set (**0.00** sec)

section A:

root@localhost : test **04**:**59**:**30**>start transaction;
Query OK, **0** rows affected (**0.00** sec)

root@localhost : test **04**:**59**:**33**>select * from t where a = **8** for update; +---+
| a |
+---+
| **8** |
+---+
**1** row in set (**0.00** sec)

section B:

root@localhost : test **04**:**58**:**41**>start transaction;
Query OK, **0** rows affected (**0.00** sec)

root@localhost : test **04**:**59**:**45**>insert into t values(**6**);
Query OK, **1** row affected (**0.00** sec)

root@localhost : test **05**:**00**:**05**>insert into t values(**7**);
Query OK, **1** row affected (**0.00** sec)

root@localhost : test **05**:**00**:**08**>insert into t values(**9**);
Query OK, **1** row affected (**0.00** sec)

root@localhost : test **05**:**00**:**10**>insert into t values(**10**);
Query OK, **1** row affected (**0.00** sec)

```

问题：

为什么section B上面的插入语句可以正常，和测试一不一样？

分析：

因为InnoDB对于行的查询都是采用了Next-Key Lock的算法，锁定的不是单个值，而是一个范围，按照这个方法是会和第一次测试结果一样。但是，当查询的索引含有唯一属性的时候，Next-Key Lock 会进行优化，将其降级为Record Lock，即仅锁住索引本身，不是范围。

如何让测试一不阻塞？可以显式的关闭Gap Lock 有2种方法：

1：把事务隔离级别改成：Read Committed，提交读、不可重复读。SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

2：修改参数：innodb\_locks\_unsafe\_for\_binlog 设置为1。