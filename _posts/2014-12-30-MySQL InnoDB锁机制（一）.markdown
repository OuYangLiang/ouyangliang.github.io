---
layout: post
title:  "MySQL InnoDB锁机制（一）"
date:   2014-12-30 22:19:16 +0800
categories: mysql
---
MySQL InnoDB一共有四种锁：共享锁（读锁，S锁）、排他锁（写锁，X锁）、意向共享锁（IS锁）和意向排他锁（IX锁）。其中共享锁与排他锁属于行级锁，另外两个意向锁属于表级锁。 

共享锁（读锁，S锁）：若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放S锁。
排他锁（写锁，X锁）：若事务T对数据对象A加上X锁，则只允许T读取和修改A，其他事务不能再对A加作何类型的锁，直到T释放A上的X锁。
意向共享锁（IS锁）：事务T在对表中数据对象加S锁前，首先需要对该表加IS（或更强的IX）锁。
意向排他锁（IX锁）：事务T在对表中的数据对象加X锁前，首先需要对该表加IX锁。
比如SELECT ... FROM T1 LOCK IN SHARE MODE语句，首先会对表T1加IS锁，成功加上IS锁后才会对数据加S锁。
同样，SELECT ... FROM T1 FOR UPDATE语句，首先会对表T1加IX锁，成功加上IX锁后才会对数据加X锁。
MySQL InnoDB 锁兼容阵列
