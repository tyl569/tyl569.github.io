---
layout: post
title: Redis数据类型之HyperLogLog
date: 2019-11-21
tags: ["Docker","HyperLogLog","Redis","统计"]
---

Redis相对于memcache的优势之一就是支持丰富的数据结构，比如Hash、List、Set、Zset等。除了这些以外，redis还支持HyperLogLog

### HyperLogLog

假如有个需求，需要统计UV情况，我们的思路是什么？

*   Hash: 我们可以使用Hash的结构，使用用户的ip当做元素的key，最后使用`HLEN`统计下个数
*   Set: Set是无序唯一的，同样可以使用用户的IP作为key，最后使用`SCARD`统计个数

没错，这两种都可以实现需求，但是对内存的占用是惊人的，如果是上千万的UV，那么会占用大量的内存。那么有没有`物美价廉`的方式呢？那就是HyperLogLog。

### HyperLogLog的优势和劣势

HyperLogLog只会占用`12KB`左右的存储空间，这个既是优势，优势劣势，因为如果数量比较小，这个`12KB`左右的空间是非常不划算的。

但是redis也对HyperLogLog进行了优化，在计数比较小的时候，采用稀疏矩阵存储，占用的空间比较小，只有当超过了某个阈值，才会一次性变得稠密，才会占用`12KB`.

HyperLogLog的劣势就是会出现统计的误差，并不能精确的进行个数统计.

### 对比情况

    ##伪代码
    <?php
    $redisObj = new Redis();

    add2HyperLogLog();
    add2Hash();
    add2Set();

    function add2HyperLogLog()
    {
        global $redisObj;
        $connect = $redisObj::getConn();
        for ($i = 0; $i < 100000; $i++) {
            $user["user_name_".$i] = "user_" . $i;
        }
        $connect->pfAdd("user_by_hyper_log_log", $user);
    }

    function add2Hash()
    {
        global $redisObj;
        $redisObj = new \Lta\Redis();
        $connect = $redisObj::getConn();
        for ($i = 0; $i < 100000; $i++) {
            $connect->sAdd("user_by_set", "user_" . $i);
        }
    }

    function add2Set()
    {
        global $redisObj;
        $redisObj = new \Lta\Redis();
        $connect = $redisObj::getConn();
        for ($i = 0; $i < 100000; $i++) {
            $connect->hSet("user_by_hash", "user_name_".$i,"user_" . $i);
        }
    }

    10.188.40.78:6379> DEBUG OBJECT user_by_hash
    Value at:0x7fc170333f10 refcount:1 encoding:hashtable serializedlength:2677785 lru:14064726 lru_seconds_idle:147
    10.188.40.78:6379> DEBUG OBJECT user_by_hyper_log_log
    Value at:0x7fc1700a6000 refcount:1 encoding:raw serializedlength:10592 lru:14064728 lru_seconds_idle:150
    10.188.40.78:6379> DEBUG OBJECT user_by_set
    Value at:0x7fc1700a6010 refcount:1 encoding:hashtable serializedlength:1088895 lru:14064734 lru_seconds_idle:150

    10.188.40.78:6379> PFCOUNT user_by_hyper_log_log
    (integer) 99839
    10.188.40.78:6379> SCARD user_by_set
    (integer) 100000
    10.188.40.78:6379> HLEN user_by_hash
    (integer) 100000

<table>
<thead>
<tr>
<th>键名</th>
<th>长度</th>
<th>元素个数</th>
</tr>
</thead>
<tbody>
<tr>
<td>user_by_hash</td>
<td>2677785</td>
<td>100000</td>
</tr>
<tr>
<td>user_by_hyper_log_log</td>
<td>10592</td>
<td>99839</td>
</tr>
<tr>
<td>user_by_set</td>
<td>1088895</td>
<td>100000</td>
</tr>
</tbody>
</table>

可以看到，使用user_by_hyper_log_log的存储，长度要小很多，但是统计的元素格式是不完整的，误差率是`0.161%`，对于统计UV来说，是可以接受的。

### 使用rdbtools

但是serializedlength并不是真实的占用空间，并且在存储的时候，可能会进行序列化，要想查看真实的空间，需要使用另外的工具

    $ pip install rdbtools
    Successfully built rdbtools
    Installing collected packages: rdbtools
    Successfully installed rdbtools-0.1.14

    $ redis-memory-for-key -s 10.188.40.78 user_by_hyper_log_log
    Key             user_by_hyper_log_log
    Bytes               14400
    Type                string
    $ redis-memory-for-key -s 10.188.40.78 user_by_hash
    Key             user_by_hash
    Bytes               7892932.0
    Type                hash
    Encoding            hashtable
    Number of Elements      100000
    Length of Largest Element   15
    $ redis-memory-for-key -s 10.188.40.78 user_by_set
    Key             user_by_set
    Bytes               5572932.0
    Type                set
    Encoding            hashtable
    Number of Elements      100000
    Length of Largest Element   10

这个对比结果就很明显了，Hash占用的空间是HyperLogLog的548倍，Set占用的空间是HyperLogLog的387倍！ 

换算下占用的HyperLogLog的占用空间，大概是`14KB`

### 使用场景

前面有说道，HyperLogLog是存在误差的，一般是一些对可接受小误差的统计，比如：

*   统计注册 IP 数
*   统计每日访问 IP 数
*   统计页面实时 UV 数
*   统计在线用户数
*   统计用户每天搜索不同词条的个数

### 参考文献：

*   《Redis深度历险》--钱文品
*   [Redis：HyperLogLog使用与应用场景](https://blog.csdn.net/maoyuanming0806/article/details/81814610)

本文链接： [https://feilong.tech/2019/11/21/redis-hyperloglog](https://feilong.tech/2019/11/21/redis-hyperloglog)