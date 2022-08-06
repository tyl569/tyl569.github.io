---
title: MySQL隐式record lock 锁
date: 2022-07-20 09:23:28
tags:
    - MySQL
    - 锁
categories:
  - - Docker
  - - Mysql
---

MySQL在进行数据更新的时候，为了防止并发问题，会在更新的时候加上一个隐式的record lock锁。

<!--more-->

我们新建一张表，然后插入几条测试数据

```sql
CREATE TABLE t(id INT UNSIGNED PRIMARY KEY ,NAME VARCHAR(100));
INSERT INTO t VALUES(1,'a'),(4,'b'),(7,'c'),(10,'d'),(20,'e'),(30,'f');
```

| id | NAME |
|----|------|
| 1 | a    |
| 4 | b    |
| 7 | c    |
| 10| d    |
| 20| e    |
| 30| f    |


接下来我们开启两个事务，看下锁的情况

|session1|session2|
|-------|-------|
|START TRANSACTION;||
|UPDATE t SET `name`=b WHERE id=1;        |       |
|                |     START TRANSACTION;|
||SELECT * FROM t  WHERE id=1;  |
||UPDATE t SET `name`='b' WHERE id=1;  sql无法执行，数据被锁|

所有可以看出来，隐式的record lock是共享锁，而不是排他锁。

