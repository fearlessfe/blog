---
title: "Mysql Lock"
date: 2022-07-19T21:09:03+08:00
draft: false
---

## 锁类型

mysql 中的锁大致分为 全局锁，表级锁和行锁 三类

### 全局锁

全局锁就是对整个数据库实例加锁，命令如下

Flush tables with read lock (FTWRL)

全局锁的典型使用场景就是做全库逻辑备份

### 表级锁

mysql 表级锁也分为 表锁和元数据锁

表锁的语句如下

lock tables ...read/write eg: lock tables t1 read, t2 write

执行上面的语句后，所有线程均不可 读t1，写t2

元数据锁主要是对表结构锁定，防止更新数据的过程中更改表的结构。
元数据锁不用显示声明，访问一个表的时候会自动加索

### 行锁

MySQL的行锁是由各个引擎实现的

行锁，innodb 只有通过检索索引才会有行锁

不同事务中，对相同的行的更新会引起锁的争抢，甚至会引起死锁。
