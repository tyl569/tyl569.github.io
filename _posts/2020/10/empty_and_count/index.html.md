---
layout: post
title: empty和count哪个性能会更好？
date: 2020-10-25
tags: ["PHP","PHP","PHP7","PHP源码","PHP源码"]
---

### 疑问

有些事情其实是比较让我感到疑惑的，就是关于使用empty和count函数，对数组判空，哪个性能会更好？

一般来说，我们对数组判空，常用的就是empty和count。即：

    if (empty($arr)) {

    }

    if (count($arr) == 0) {

    }

### 论证

是的，这两种都能实现，但是哪种性能会更好呢？所以，我做了简单对比：

    <?php

    $arr = array_fill(0, 100, 1);
    $startTime = markTime();
    for ($i = 0; $i < 10000000; $i++) {
        if (empty($arr)) {

        }
    }
    $endTime = markTime();
    echo "使用empty判空花了: " . (floatval($endTime) - floatval($startTime)) . "\n";

    $startTime = markTime();
    for ($i = 0; $i < 10000000; $i++) {
        if (count($arr) == 0) {

        }
    }
    $endTime = markTime();
    echo "使用count判空花了: " . (floatval($endTime) - floatval($startTime)) . "\n";

    function markTime()
    {
        list($mic, $sec) = explode(" ", microtime());
        return $mic + $sec;
    }

上面的例子，我通过循环1kw次，计算前后执行的时间情况，得到了一下数据：

    使用empty判空花了: 0.15395402908325
    使用count判空花了: 0.2148220539093

我担心某次的试验结果不准确，所以进行了多次了试验，试验结果都是一样的，那就是`empty`比`count`性能更好。

### 分析

对比下opcode:

    php7 -dvld.active=1 test1.php
    Finding entry points
    Branch analysis from position: 0
    1 jumps found. (Code = 62) Position 1 = -2
    filename:       /Users/feilong/data/service/i.api-big.crep.ke.com/test1.php
    function name:  (null)
    number of ops:  4
    compiled vars:  none
    line     #* E I O op                           fetch          ext  return  operands
    -------------------------------------------------------------------------------------
       3     0  E >   INIT_FCALL                                               'count'
             1        SEND_VAL                                                 <array>
             2        DO_ICALL                                                 
             3      > RETURN                                                   1

    branch: #  0; line:     3-    3; sop:     0; eop:     3; out0:  -2
    path #1: 0,

    php7 -dvld.active=1 count.php
    Finding entry points
    Branch analysis from position: 0
    1 jumps found. (Code = 62) Position 1 = -2
    filename:       /Users/feilong/data/service/test/count.php
    function name:  (null)
    number of ops:  4
    compiled vars:  none
    line     #* E I O op                           fetch          ext  return  operands
    -------------------------------------------------------------------------------------
       3     0  E >   INIT_FCALL                                               'count'
             1        SEND_VAL                                                 <array>
             2        DO_ICALL                                                 
             3      > RETURN                                                   1

    branch: #  0; line:     3-    3; sop:     0; eop:     3; out0:  -2
    path #1: 0,

    php7 -dvld.active=1 empty.php
    Finding entry points
    Branch analysis from position: 0
    1 jumps found. (Code = 62) Position 1 = -2
    filename:       /Users/feilong/data/service/test/empty.php
    function name:  (null)
    number of ops:  1
    compiled vars:  none
    line     #* E I O op                           fetch          ext  return  operands
    -------------------------------------------------------------------------------------
       3     0  E > > RETURN                                                   1

    branch: #  0; line:     3-    3; sop:     0; eop:     0; out0:  -2
    path #1: 0, 

通过opcode，可以清晰看出，count的函数，生成的opcode总共是4行`INIT_FCALL`，`SEND_VAL`,`DO_ICALL`, `RETURN`；而empty的opcode只有`RETURN`。这是因为count是使用拓展方式，将函数加载到PHP内核，所以在执行的时候，会进行一些模块的初始化操作，而empty是在代码扫描的阶段，就已经进行了加载。所以执行效率会更高。

本文链接： [https://feilong.tech/2020/10/25/empty_and_count/](https://feilong.tech/2020/10/25/empty_and_count/)