---
title: MySQL之虚拟列（generated-columns）
tags:
  - MySQL
id: '458'
categories:
  - - Mysql
date: 2019-08-14 23:50:01
---

### 定义

MySQL虚拟列（generated-columns）是MySQL 5.7加入的新特性。怎么理解虚拟列？从名字来讲，“生成的字段”，并不是主动插入的值。 MySQL的文档，是这么解释虚拟列的：

> There are two kinds of Generated Columns: virtual (default) and stored. Virtual means that the column will be calculated on the fly when a record is read from a table. Stored means that the column will be calculated when a new record is written in the table, and after that it will be treated as a regular field. Both types can have NOT NULL restrictions, but only a stored Generated Column can be be a part of an index.
> 
> 解释起来，就是虚拟列支持两种方式，virtual和stored。当在表里读取记录的时候，virtual类型的会进行实时的计算。当写入一条记录的时候，stored类型会通过计算，写入表中，和常规的字段的一样的占用磁盘的空间。这两种类型都可以有NOT NULL限制，但是能使用索引的一部分的功能。

<!--more-->

MySQL的官方，提供了一个例子，用来简单的说明虚拟列的作用。

```sql
> CREATE TABLE sales(
name VARCHAR(20),
price_eur DOUBLE,
amount INT,
total_eur DOUBLE AS (price_eur * amount),
total_used DOUBLE AS (total_eur * xrate),
xrate DOUBLE);
> INSERT INTO sales(name,price_eur,amount,xrate) VALUES('尺子', 1.2, 10, 0.9);
> SELECT * FROM sales;
nameprice_euramounttotal_eurtotal_usedxrate
尺子1.2101210.80.9
```

这个例子应该算是比较明了了，total\_eur和total\_used根据计算的公式，自动计算除了结果。

### 使用的场景

虚拟列的使用场景其实还算是挺多的，就想上面的例子，可以计算一些公式。尤其对一致性要求比较高的。如果每次都是通过代码进行计算，可能会由于人为的原因，某个字段的计算结果，没有update，那么就会产生bug。如果使用虚拟列，那么直接更新比较的值就好，没必要更新计算结果，降低的人为误操作的风险。

#### 实时计算

举个例子，我们可能会需要记录三角形的三边，即：两个直角边，和一个斜边。

按照一般的逻辑，我们可能会，通过代码直接进行计算

```php
$a = 4;
$b = 3;
$c = sqrt(pow($a, 2) + pow($b, 2));
// 插入到数据库
```

这样做是可以的，但是可能会由于人为原因，导致计算的步骤有问题，比如由于人为疏忽，导致忘记了把斜边的值更新到数据库。

我们可以通过MySQL创建一个斜边的虚拟列，然后自动进行计算。

```sql
>  CREATE TABLE `triangle` (
 `sidea` double DEFAULT NULL,
 `sideb` double DEFAULT NULL,
 `sidec` double GENERATED ALWAYS AS (SQRT(sidea * sidea + sideb * sideb))
 ) ;
```

#### 数据冗余

这个场景也是比较常见，比如，我们的某个字段存储的是json结构，但是为了方便查询，可能需要json里面的某个子单当做SQL的查询条件，这个时候，我们可以把这个查询条件，作为虚拟列。

### 使用限制

虚拟列虽然是计算的结果，但是也是有一些限制的。

#### 恶意的数据

```sql
> create table t( x int, y int, z int generated always as( x / y));
insert into t(x,y) values(1,0); 
1365 - Division by 0, Time: 0.043000s
```

根据创建的表语句，z是x和y的商，由于要插入的值y=0，导致计算的时候出现了错误。

#### 删除源数据的列

还是以第一个表的数据为例

```sql
> CREATE TABLE sales(
name VARCHAR(20),
price_eur DOUBLE,
amount INT,
total_eur DOUBLE AS (price_eur * amount),
total_used DOUBLE AS (total_eur * xrate),
xrate DOUBLE);
> alter table sales drop price_eur;
3108 - Column 'price_eur' has a generated column dependency.
```

#### 索引的限制

虚拟列是不允许创建主键索引和全文索引的。

```sql
> CREATE TABLE sales(
name VARCHAR(20),
price_eur DOUBLE,
amount INT,
total_eur DOUBLE AS (price_eur * amount),
total_used DOUBLE AS (total_eur * xrate),
total_used2 DOUBLE AS (total_eur * xrate) stored,
xrate DOUBLE);
```

我重新创建了一个表，下面来看看virtual和stored在索引上的区别吧。

```sql
> ALTER TABLE sales ADD PRIMARY KEY(total_eur);
3106 - 'Defining a virtual generated column as primary key' is not supported for generated columns.
> ALTER TABLE sales ADD PRIMARY KEY(total_used2);
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

第一个区别就是，virtual是不允许作为主键的，这大概是因为virtual是实时计算的值，并且并没有写到磁盘上，没办法使用聚集索引。

```sql
> ALTER TABLE sales ADD fulltext index (total_eur);
3106 - 'Fulltext index on virtual generated column' is not supported for generated columns.
> ALTER TABLE sales ADD fulltext index(total_used2);
1283 - Column 'total_used2' cannot be part of FULLTEXT index
```

很明显，两者都不能创建全文索引。

### 总结

虚拟列是MySQL 5.7版本之后新增的特性，主要是方便我们查询和操作。

我所说的可能只是冰山一角，具体的用法，还需要我们自己根据具体的业务场景使用才行。

### 参考文献

*   [MySQL 5.7新特性之Generated Column](https://www.linuxidc.com/Linux/2016-02/128066.htm)
    
*   [Generated Columns in MySQL 5.7.5](http://mysqlserverteam.com/generated-columns-in-mysql-5-7-5/)
    

本文连接：[http://feilong.tech/2019/08/14/mysql-generated-columns](http://feilong.tech/2019/08/14/mysql-generated-columns)