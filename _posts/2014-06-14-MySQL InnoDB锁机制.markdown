---
layout: post
title:  "MySQL InnoDB锁机制"
date:   2014-06-14 22:19:16 +0800
categories: mysql
keywords: mysql,隔离级别,索引,行锁,innodb
description: 介绍mysql封锁机制
commentId: 2014-06-14
---
在我们的日常工作中，经常会遇到各种死锁的场景，有的死锁分析起来是比较容易的，比如同类型的事务并发引起的死锁；而不同类型事务并发引起的死锁，分析起来就不是那么容易了。系统化的了解数据库的加锁机制，不仅有助于对现有问题的分析，在设计阶段也能更好的把握系统的性能与复杂业务场景的解决方案。

要系统化了了解数据库的加锁机制，我们需要弄明白：

* 有哪些类型的锁
* 对什么东西加锁
* 以什么样的方式加锁

## 1. 有哪些类型的锁

MySQL InnoDB一共有四种锁：共享锁（读锁，S锁）、排他锁（写锁，X锁）、意向共享锁（IS锁）和意向排他锁（IX锁）。其中共享锁与排他锁属于行级锁，另外两个意向锁属于表级锁。

* **共享锁（读锁，S锁）**：

    若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放S锁。

* **排他锁（写锁，X锁）**：

    若事务T对数据对象A加上X锁，则只允许T读取和修改A，其他事务不能再对A加作何类型的锁，直到T释放A上的X锁。

* **意向共享锁（IS锁）**：

    事务T在对表中数据对象加S锁前，首先需要对该表加IS（或更强的IX）锁。

* **意向排他锁（IX锁）**：

    事务T在对表中的数据对象加X锁前，首先需要对该表加IX锁。

比如`SELECT ... FROM T1 LOCK IN SHARE MODE`语句，首先会对表T1加IS锁，成功加上IS锁后才会对数据加S锁。

同样，`SELECT ... FROM T1 FOR UPDATE`语句，首先会对表T1加IX锁，成功加上IX锁后才会对数据加X锁。

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<caption>MySQL InnoDB 锁兼容阵列</caption>
<tr class="info"><td></td><td>X</td><td>IX</td><td>S</td><td>IS</td></tr>
<tr><td class="info">X</td><td>✗</td><td>✗</td><td>✗</td><td>✗</td></tr>
<tr><td class="info">IX</td><td>✗</td><td>✓</td><td>✗</td><td>✓</td></tr>
<tr><td class="info">S</td><td>✗</td><td>✗</td><td>✓</td><td>✓</td></tr>
<tr><td class="info">IS</td><td>✗</td><td>✓</td><td>✓</td><td>✓</td></tr>
</table>
</div>
</div>

MySQL官网上有个死锁的例子，但分析得过于概括，这里我们详细分析一下。

首先，会话S1以SELECT * FROM t WHERE i = 1 LOCK IN SHARE MODE查询，该语句首先会对t表加IS锁，接着会对数据（i = 1）加S锁。

```shell
mysql> CREATE TABLE t (i INT) ENGINE = InnoDB;  
Query OK, 0 rows affected (1.07 sec)  

mysql> INSERT INTO t (i) VALUES(1);  
Query OK, 1 row affected (0.09 sec)  

mysql> START TRANSACTION;  
Query OK, 0 rows affected (0.00 sec)  

mysql> SELECT * FROM t WHERE i = 1 LOCK IN SHARE MODE;  
+------+  
| i    |  
+------+  
|    1 |  
+------+  
1 row in set (0.10 sec)  
```

接着，会话S2执行DELETE FROM t WHERE i = 1，该语句尝试对t表加IX锁，由于IX锁与IS锁是兼容的，所以成功对t表加IX锁。接着继续对数据（i = 1）加X锁，但数据已经被会话S1事务加了S锁了，所以会话S2等待。

```shell
mysql> START TRANSACTION;  
Query OK, 0 rows affected (0.00 sec)  

mysql> DELETE FROM t WHERE i = 1;  
```

接着，会话S1也执行DELETE FROM t WHERE i = 1，该语句首先对t表加IX锁，虽然会话S2已经对t表加了IX锁，但IX锁与IX锁之间是兼容的，所以成功对t表加上IX锁。接着会话S1会对数据（i = 1）加X锁，此时发现会话S2占有的IX锁与X锁不兼容，所以会话S1也等待。就这样，会话S1等S2释放IX锁，而会话S2等S1释放S锁，从而死锁发生。

```shell
mysql> DELETE FROM t WHERE i = 1;  
Query OK, 1 row affected (0.00 sec)  

mysql>  
```

上例中会话S1虽然执行成功了，但是下面会话S2发生了死锁。这是因为Mysql检测到死锁后，会自动强制其中一个事务退出。

```shell
mysql> DELETE FROM t WHERE i = 1;  
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction  
mysql>  
```

## 2. 对什么加锁

每一张InnoDB表都有且仅有一个特殊的索引，聚族索引（Clustered Index），表中的数据是直接存放在聚族索引的叶子节点中，这样，根据聚族索引查询就会比普通索引更快，因为少了一次IO操作。

通常，**聚族索引就是表的主键；如果表没有主键，那InnoDB会把第一个非空的唯一索引当作聚族索引；如果表既无主键，又无非空的唯一索引，那么InnoDB会创建一个隐藏的索引作为聚族索引。表中的其它索引，都叫做第二索引（Secondary Index），第二索引中只包含自身索引列和聚族索引列的内容，所以当一个表的主键很长时，其它的索引都会受到影响**。

为什么要先讲聚族索引呢？因为这对理解InnoDB加锁机制很重要，**InnoDB加锁的对象不是返回的数据记录，而是查询这些数据时所扫描过的索引。当我们执行一个锁读（SELECT ... LOCK IN SHARE MODE或者SELECT ... FOR UPDATE）时，InnoDB不是对最终的返回结果加锁，而是对查询这些结果时所扫描的索引加锁，如果被扫描的索引不是聚族索引，那被扫描的索引以及它所指向的聚族索引也会被加锁**。由此可知，当一个锁读无法使用索引的话，InnoDB就是遍历整个表（遍历整个聚族索引），从而把整张表都锁住。

## 3. 以什么样的方式加锁

MySQL InnoDB支持三种行锁定方式：

* **行锁（Record Lock）**：

    锁直接加在索引上。

* **间隙锁（Gap Lock）**：

    锁加在不存在的空闲空间，可以是两个索引记录之间，也可能是第一个索引记录之前或最后一个索引之后的空间。

* **Next-Key Lock**：
    行锁与间隙锁组合起来用就叫做Next-Key Lock。

默认情况下，InnoDB工作在`可重复读`隔离级别下，并且以Next-Key Lock的方式对数据行进行加锁，这样可以有效防止幻读的发生。Next-Key Lock是行锁与间隙锁的组合，这样，当InnoDB扫描索引记录的时候，会首先对选中的索引记录加上行锁（Record Lock），再对索引记录两边的间隙加上间隙锁（Gap Lock）。如果一个间隙被事务T1加了锁，其它事务是不能在这个间隙插入记录的。

还是用之前的员工表举例：

```shell
+----+------+--------+--------+
| id | num  | depart | name   |
+----+------+--------+--------+
| 10 | 1010 |   5100 | 张三   |
| 20 | 1020 |   5200 | 李四   |
| 30 | 1030 |   5300 | 王五   |
| 40 | 1040 |   5100 | 刘大   |
+----+------+--------+--------+
```

其中，id表示记录主键；num表示员工工号，唯一索引；depart表示员工所在的部门编号，普通索引；name表示员工姓名。测试前插入一些测试数据。

depart索引可以表示为下面一张二维表：

<div class="row">
<div class="col-sm-6">
<table class="table table-bordered table-condensed table-striped text-center">
<caption>depart索引顺序二维表示</caption>
<tr><td class="info">depart</td><td>5100</td><td>5100</td><td>5200</td><td>5300</td></tr>
<tr><td class="info">id</td><td>10</td><td>40</td><td>20</td><td>30</td></tr>
</table>
</div>
</div>

由上面的表可知，depart索引将整个间隙拆分为五个区间：

    (-∞, [5100|10])，([5100|10], [5100|40]), ([5100|40], [5200|20]), ([5200|20], [5300|30])与([5300|30], +∞)

现在我们在`可重复读`级别下，根据索引字段depart = 5100加锁查询的话，应该会锁定`(-∞, [5100|10])，([5100|10], [5100|40]), ([5100|40], [5200|20])`这三个间隙

```shell
mysql> set autocommit=0;
Query OK, 0 rows affected (0.01 sec)

mysql> set session transaction isolation level repeatable read;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from employee where depart = 5100 lock in share mode;
+----+------+--------+--------+
| id | num  | depart | name   |
+----+------+--------+--------+
| 10 | 1010 |   5100 | 张三   |
| 40 | 1040 |   5100 | 刘大   |
+----+------+--------+--------+
2 rows in set (0.00 sec)
```

另起一个会话测试一下：

```shell
mysql> insert into employee values (1, 9999, 5000, 'xx');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into employee values (15, 9999, 5100, 'xx');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into employee values (55, 9999, 5100, 'xx');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into employee values (55, 9999, 5150, 'xx');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into employee values (15, 9999, 5200, 'xx');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> insert into employee values (25, 9999, 5200, 'xx');
Query OK, 1 row affected (0.00 sec)

mysql> select * from employee where id = 10 for update;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> select * from employee where id = 40 for update;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql> select * from employee where depart = 5100 for update;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
mysql>
```

其中(5000,1)是在间隙`(-∞, [5100|10])`，(5100,15)是在间隙`([5100|10], [5100|40])`，(5100,55)、(5150,55)与(5200,15)在间隙`([5100|40], [5200|20])`，所以前四个插入都锁等待超时。

间隙锁在InnoDB的唯一作用就是防止其它事务的插入操作，以此来达到防止幻读的发生，所以间隙锁不分什么共享锁与排它锁。
另外，在上面的例子中，我们选择的是一个普通（非唯一）索引字段来测试的，这不是随便选的，因为如果InnoDB扫描的是一个主键、或是一个唯一索引的话，那InnoDB只会采用行锁方式来加锁，而不会使用Next-Key Lock的方式，也就是说不会对索引之间的间隙加锁，大家可以想想为什么？

上例中后面三个查询超时，表示索引`depart = 5100`，及它所指向的聚族索引`id = 10 & id = 40`都被加了读锁（行锁）。

间隙锁存在的意义是为了解决幻读的问题，在`读已提交`级别下，InnoDB是不会使用间隙锁的，因为这个级别本身不要求避免幻读的发生，所以在`读已提交`级别下，InnoDB只会以行锁的方式对索引加锁，不会使用间隙索。

## 总结

1. SELECT * FROM ... LOCK IN SHARE MODE对索引加共享锁；

    SELECT * FROM ... FOR UPDATE对索引加排他锁。

2. UPDATE与DELETE语句对索引加排他锁。
3. INSERT INTO ... 对间隙加意向插入锁（相当于范围为1的间隙锁）。
4. SELECT * FROM ... 是非阻塞式读，（除Serializable级别）不会对索引加锁。在读已提交级别下，总是查询记录的最新、有效的版本；在可重复读级别下，会记住第一次查询时的版本，之后的查询会基于该版本。例外的情况是在Serializable级别，这时会以Next-Key Lock的方式对索引加锁。
5. 在可重复读级别下，InnoDB以Next-Key Lock的方式对索引加锁；在读已提交级别下，InnoDB以Index-Record Lock的方式对索引加锁。
6. 被加锁的索引如果不是聚族索引，那索引所指向的聚族索引也会被加锁（如果是索引覆盖查询，则不会对聚族索引加锁）。
