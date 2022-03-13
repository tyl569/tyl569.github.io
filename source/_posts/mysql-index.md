---
title: 索引对性能到底有多少的影响？？
tags: []
id: '274'
categories:
  - - Linux
  - - Mysql
comments: false
date: 2018-04-13 20:06:33
---

#### 索引到底对性能有多少影响？

这个问题估计是很多MySQL小白好奇的问题。当然我也是一样。因为之前的时候，并没有对索引有太多的注意，而且之前的工作经历，因为数据量很小，索引所起到的作用并不是很大，所以也没有太大注意。

<!--more-->

#### 事情的起点

我在公司是做后端开发（PHPer），除了日常的开发工作，也要兼职公司的运维。每周安排一个人跟进报警邮件，出现问题及时通报。

像很多创业公司的一样，我们选用的是阿里云的ECS+RDS。因为如果自己购买服务器，不管是运维成本还是物理成本都是比较高的。

一天将近半夜12点的时候，报警日志突然出现了`MySQL server has gone away`

遇到问题肯定是先Baidu，我找到了MySQL官方的解释，原因是查询的时候，出现的mysql断开的情况。我登录阿里云rds后台，发现wait\_timeout时间长得很。不应该会出现超时的情况。

![](/uploads/2018/04/29826BAFF73C1D32CDD9C5A89502CB29.png)

一个同事：“会不会和rds经常CPU报警有关？”

我勒个去，我查了一下rds监控，果然CPU持续升高。

![](/uploads/2018/04/1523616180755.jpg)

#### 问题跟进

rds自带了日志系统，可以方便。查看了一下慢日志系统，果然有很多的慢SQL日志。

![](/uploads/2018/04/WX20180413-194555.png)

我曹，每次扫描了8W多行。看来是没有使用到索引。加上次数频繁，解析的总次数高达 1762833762 行。

#### 定位问题

查看MySQL的执行计划

```sql
mysql> explain extended SELECT * FROM `test` WHERE `is_deleted` =0 AND `a` = 81644;
+----+-------------+-----------+------+---------------+------+---------+------+-------+-------------+
 id  select_type  table      type  possible_keys  key   key_len  ref   rows   Extra       
+----+-------------+-----------+------+---------------+------+---------+------+-------+-------------+
  1  SIMPLE       test  ALL   a_index       NULL  NULL     NULL  86172  Using where 
+----+-------------+-----------+------+---------------+------+---------+------+-------+-------------+
1 row in set, 3 warnings (0.01 sec)
mysql> show warnings;
 Level    Code  Message                                                                                             
 Warning  1739  Cannot use ref access on index 'a_index' due to type or collation conversion on field 'a'                                                                                                 
 Warning  1739  Cannot use range access on index 'a_index' due to type or collation conversion on field 'a'
```

果然是没有用到索引，全表扫描。

原来，由于a数据类型是varchar类型的。但是查询的时候，使用的int类型，在执行SQL语句的时候，由于类型原因，造成了隐式转换。没有用到索引。所以实际上，应该把原来的SQL语句更改成 `SELECT * FROM test WHERE is_deleted =0 AND a = '81644'`。

虽然原因找到了，但是查询的SQL那么多，定位那具体的php文件以及对于的代码行数，也是一个难题。

#### PHP慢日志

为了能够定位代码的效率，PHP自带一个功能，那就是慢日志。如果PHP脚本，执行时间比较长的时候，那么PHP会认为这段代码是有问题的，PHP会把代码的基本信息打印到慢日志里面，能够方便开发者定位问题。

这么说来，如果找到慢日志里面关于执行这个SQL的代码，也就能够准确定位到对应的PHP文件。

#### 索引对性能的影响！

接下来用对比图来比较下使用索引和没有使用索引的对比吧

![](/uploads/2018/04/WX20180413-200112.png)

优化之后的SQL执行效率，相比之前要高出很多，CPU占用率稳定保持在个位数，甚至 5%一下，相比之前80%左右，呈现指数的翻倍。

#### 总结

其实隐式转换是MySQL索引经常遇到的问题。我最开始听说是前段时间，阿里云组织了一个[慢SQL的优化大赛](https://yq.aliyun.com/roundtable/56333?utm_content=m_25986)。虽然没有得到名次，但是确实通过大赛，学到了很多关于索引的知识。