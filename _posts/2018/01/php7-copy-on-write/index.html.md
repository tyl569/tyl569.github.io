---
layout: post
title: PHP7内存管理之写时复制
date: 2018-01-21
tags: ["Linux","PHP","PHP7","PHP源码","PHP源码","内存"]
---

其实PHP的内存管理是包含[引用计数](http://feilong.tech/?p=199)和写时复制两部分，这篇文章主要是介绍写时复制。

#### 简要介绍

其实写时复制在计算机中有很多应用，它只在必要的时候才会进行深拷贝，也就是把保存的值连同内存一块拷贝一份，可以很好的节省效率。比如，Linux在fork子进程的时候，不会立刻复制父进程的地址空间，而是和父进程共享一个地址空间，只有在必要写入的时候，才会复制地址空间，和父进程进行分离。简单来讲，资源的复制是只有需要写入的时候，再回进行，再次之前，都是以只读的方式进行共享。

#### PHP的写时复制

PHP的写时复制原理是一样的。当变量要修改value的结构的时候，这个时候，就会对之前共享的内存资源进行复制一份进行修改，同事断开原来的指向，指向复制后的内存地址。
举个例子：

    <?php
    $a = array(1, 2);
    $b = $a;
    $c = $b;
    echo xdebug_debug_zval('a');
    ?>

PHP7
a: (refcount=3, is_ref=0)=array (0 => (refcount=0, is_ref=0)=1, 1 => (refcount=0, is_ref=0)=2)

    <?php
    $a = array(1, 2);
    $b = $a;
    $c = $b;
    //进行分离
    $c[] = 3;
    echo xdebug_debug_zval('a');
    ?>

PHP7
a: (refcount=2, is_ref=0)=array (0 => (refcount=0, is_ref=0)=1, 1 => (refcount=0, is_ref=0)=2)

运行结果很明显，当变量c新插入了一个元素，对那么就没有在继续引用变量a，而是独立复制了一份。

![](%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6-2.png)

然而并不是所有的类型都是支持写时复制，比如对象、资源就无法进行复制，也就是无法进行分离，如果多个变量指向同一个对象，当其中一个变量修改对象的时候，其修改将会反应到所有对象上面。事实上只有string和array两种支持分离。
举个例子：

    <?php
    class test {
            public $c = 123;
    }

    $a = new test();
    $b = $a;
    $c = $b;
    echo xdebug_debug_zval('a');
    $c->c = 456;
    echo $a->c;
    echo "\n";
    echo xdebug_debug_zval('a');
    ?>

PHP7
a: (refcount=3, is_ref=0)=class test { public $c = (refcount=0, is_ref=0)=123 }
456
a: (refcount=3, is_ref=0)=class test { public $c = (refcount=0, is_ref=0)=456 }

同样，变量a实例化了一个新的对象，然后依次进行赋值给其他变量，使用xdebug_debug_zval的时候，打印出来了变量a的3次引用计数，然后对变量c进行赋值，咦？居然发现变量a的引用计数没有变化，所以object的类型是不支持写时复制的。

<table>
<thead>
<tr>
<th style="text-align: left;">支持复制的value类型：</th>
<th style="text-align: left;">type</th>
<th>copyable</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;">simple types</td>
<td style="text-align: left;">N</td>
</tr>
<tr>
<td style="text-align: left;">string</td>
<td style="text-align: left;">Y</td>
</tr>
<tr>
<td style="text-align: left;">interned string</td>
<td style="text-align: left;">N</td>
</tr>
<tr>
<td style="text-align: left;">array</td>
<td style="text-align: left;">Y</td>
</tr>
<tr>
<td style="text-align: left;">immutable array</td>
<td style="text-align: left;">N</td>
</tr>
<tr>
<td style="text-align: left;">object</td>
<td style="text-align: left;">N</td>
</tr>
<tr>
<td style="text-align: left;">resource</td>
<td style="text-align: left;">N</td>
</tr>
<tr>
<td style="text-align: left;">reference</td>
<td style="text-align: left;">N</td>
</tr>
</tbody>
</table>

### 参考文献

《PHP7内核剖析》