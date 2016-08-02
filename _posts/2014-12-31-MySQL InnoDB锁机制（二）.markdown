---
layout: post
title:  "MySQL InnoDB锁机制（二）"
date:   2014-12-31 22:19:16 +0800
categories: mysql
---
上一篇文章我们提到MySQL InnoDB对数据行的锁定类型一共有四种：共享锁（读锁，S锁）、排他锁（写锁，X锁）、意向共享锁（IS锁）和意向排他锁（IX锁），今天我们要讨论的是MySQL InnoDB对数据行的锁定方式。

MySQL InnoDB支持三种行锁定方式：
行锁（Record Lock）：锁直接加在索引记录上面。
间隙锁（Gap Lock）：锁加在不存在的空闲空间，可以是两个索引记录之间，也可能是第一个索引记录之前或最后一个索引之后的空间。
Next-Key Lock：行锁与间隙锁组合起来用就叫做Next-Key Lock。
默认情况下，InnoDB工作在可重复读隔离级别下，并且以Next-Key Lock的方式对数据行进行加锁，这样可以有效防止幻读的发生。Next-Key Lock是行锁与间隙锁的组合，这样，当InnoDB扫描索引记录的时候，会首先对选中的索引记录加上行锁（Record Lock），再对索引记录两边的间隙加上间隙锁（Gap Lock）。如果一个间隙被事务T1加了锁，其它事务是不能在这个间隙插入记录的。
 
我们来看看例子，首先建一张表。
