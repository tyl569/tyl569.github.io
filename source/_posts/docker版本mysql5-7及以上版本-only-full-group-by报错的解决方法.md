---
title: Docker版本Mysql5.7及以上版本 ONLY_FULL_GROUP_BY报错的解决方法
tags:
  - docker
  - MySQL
id: '812'
categories:
  - - Docker
  - - Mysql
date: 2021-10-04 16:26:24
---

## 背景

最近开发的时候，需要使用MySQL的数据库，在使用group by的时候，生产环境使用的是5.6版本，但是开发机上面装的docker版本是5.7，在调用接口的时候，发现报错了，通过查询对应的资料，是因为mysql 5.7版本，默认开启了`ONLY_FULL_GROUP_BY`，所以在使用group by的时候，不能存在多余的字段信息。

## 现象回顾

### 表结构准备

```mysql
CREATE TABLE `user` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `sex` varchar(255) DEFAULT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

|id|name|sex|age|
|:---|:---|:---|:---|
|1|张三|男|21|
|2|李四|男|20|
|3|小花|女|21|

执行SQL

```mysql
SELECT * FROM `user` GROUP BY age
```

出现报错`1055 - Expression #1 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'test.user.id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by, Time: 0.064000s`

### 原因分析

MySQL的官方文档，给出如下的解释:

> MySQL 5.7.5 and later implements detection of functional dependence. If the [`ONLY_FULL_GROUP_BY`](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_only_full_group_by) SQL mode is enabled (which it is by default), MySQL rejects queries for which the select list, `HAVING` condition, or `ORDER BY` list refer to nonaggregated columns that are neither named in the `GROUP BY` clause nor are functionally dependent on them. (Before 5.7.5, MySQL does not detect functional dependency and [`ONLY_FULL_GROUP_BY`](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_only_full_group_by) is not enabled by default. For a description of pre-5.7.5 behavior, see the [MySQL 5.6 Reference Manual](https://dev.mysql.com/doc/refman/5.6/en/sql-mode.html).)

官方大致的意思是说，在5.7.5版本之后，将会开启`ONLY_FULL_GROUP_BY`，开启此配置之后，在select、having或者order by的时候，将拒绝使用非聚合列的查询。

针对上述的SQL，也就是在select+group by的时候，只能查询与group by相关列的查询。

## 解决办法

### 方法一：优化SQL

其实个人觉得，最好的办法，就是优化SQL，剔除掉无关的查询操作，将与group by相关的查询去掉:

`SELECT count(1), age FROM user GROUP BY age`

### 方法二：更改配置文件

将容器的内的配置文件，拷贝到宿主机，挂接映射关系，然后在mysqld下增加sql\_mode的配置 `sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION`

docker-compose.yml

```yaml
version: '3.1'

services:

  db:
    image: mysql
    command: --default-authentication-plugin=mysql_native_password
    restart: always
    volumes:
      - /root/docker-mysql/conf/mysql:/etc/mysql
      - /root/docker-mysql/mysql:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
    container_name: test-mysql
    ports:
      - 3307:3306
```

my.cnf

```
[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
secure-file-priv= NULL
# Custom config should go here
!includedir /etc/mysql/conf.d/
```

重启容器，查看效果

```mysql
mysql> SELECT @@sql_mode;
+----------------------------------------------------------------------------------------------------+
 @@sql_mode                                                                                         
+----------------------------------------------------------------------------------------------------+
 STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION 
+----------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> SELECT * FROM user GROUP BY age;
+----+--------+------+------+
 id  name    sex   age  
+----+--------+------+------+
  1  张三  男     21 
  2  李四  男     20 
+----+--------+------+------+
2 rows in set (0.00 sec)
```

### 方法三：更改启动命令

docker-compose.yml

```yaml
version: '3.1'

services:

  db:
    image: mysql
    command: mysqld --sql_mode="STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION" --default-authentication-plugin=mysql_native_password
    restart: always
    volumes:
      - /root/docker-mysql/conf/mysql:/etc/mysql
      - /root/docker-mysql/mysql:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
    container_name: test-mysql
    ports:
      - 3307:3306
```

销毁容器：`docker-compose down` 重启容器：`docker-compose up -d`

查看效果：

```mysql
mysql> SELECT @@sql_mode;
+----------------------------------------------------------------------------------------------------+
 @@sql_mode                                                                                         
+----------------------------------------------------------------------------------------------------+
 STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION 
+----------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> set names utf8;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> SELECT * FROM user GROUP BY age;
+----+--------+------+------+
 id  name    sex   age  
+----+--------+------+------+
  1  张三  男     21 
  2  李四  男     20 
+----+--------+------+------+
2 rows in set (0.00 sec)
```