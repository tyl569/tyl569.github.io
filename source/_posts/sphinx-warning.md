---
title: 'sphinx 生成索引时出现WARNING: attribute ''''id'''' not found - IGNORING'
tags: []
id: '140'
categories:
  - - Linux
date: 2017-08-24 20:18:45
---

#### 公司由于业务发展需要使用sphinx、之前是师傅负责配置。现在变成是我来维护，由于对sphinx了解并不多，所以在生成索引的时候一直出现WARNING: attribute 'id' not found - IGNORING

<!-- more -->

我的表结构

```sql
CREATE TABLE `user` (
  `id` int(10) NOT NULL,
  `name` varchar(20) DEFAULT NULL,
  `sex` tinyint(4) DEFAULT NULL,
  `create_time` int(10) DEFAULT NULL,
  `update_time` int(10) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=gbk;
```

部分sphinx配置

```roboconf
sql_query               = \
                SELECT id, name, sex, create_time, update_time FROM user
sql_attr_uint           = id
sql_attr_timestamp      = create_time
sql_attr_timestamp      = update_time
```

#### 查了一下资料，才知道sphinx会内置一个id 的字段，所以选择索引的时候，就不能设置`sql_attr_uint = id`，要不然就是重复了。但是sql第一列还必须是id字段才行 （听着有点矛盾）

#### 可以有两个思路：

##### 第一种

*   更改表结构，把id->uid
*   更改配置
    
    ```roboconf
    sql_query               = \
                SELECT uid as id, uid, name, sex, create_time, update_time FROM user
    sql_attr_uint           = uid
    sql_attr_timestamp      = create_time
    sql_attr_timestamp      = update_time
    ```
    

##### 第二种

*   不改表结构，只更改配置
    
    ```roboconf
    sql_query               = \
                SELECT  id, id as uid, name, sex, create_time, update_time FROM user
    sql_attr_uint           = uid
    sql_attr_timestamp      = create_time
    sql_attr_timestamp      = update_time
    ```
    

#### 然后再生成索引应该就好了

#### 附上sphinx配置说明

![](/uploads/2017/08/1491118645070.png) ![](/uploads/2017/08/1491118649031.png)