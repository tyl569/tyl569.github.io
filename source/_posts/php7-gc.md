---
title: PHP7内存管理之垃圾回收
tags:
  - PHP7垃圾回收
  - PHP源码
id: '205'
categories:
  - - PHP源码
comments: false
date: 2018-02-21 18:54:26
---

#### 回收过程

在自动GC机制中，在zval断开value指向的时候如果发现refcount=0的时候，则会直接释放value，这就是自动回收GC的过程。发生断开的两种情况为修改变量与函数返回的时候，修改变量的时候，会断开原有的value指向，函数返回的时候，则会释放局部变量，也就是把所有局部变量的refcount计数-1。 此外，当使用unset函数的时候，也会主动销毁这个变量。

#### 垃圾回收

虽然有了自动GC机制，但是有一种情况是没办法解决的，那就是因为变量因为循环引用而无法回收造成的内存泄露，这种情况通常是循环引用。简单来讲，循环引用就是引用自身，这种情况一般只会发生在数组或者对象的身上。比如定义了`$a = array()` ，插入一个新元素，这个元素对数组自身进行引用`$a[] = &$a`，当所有的外部引用都断开了，但是数据的refcount仍然大于0而得不到释放，但是事实上，这个变量没有在使用的价值了。

```php
<?php
$a = array();
$a[] = &$a;
unset($a);
?>
```

在unset之前，变量a是有两次引用的，一个来自$a，一个来自$a\[1\]

![](/uploads/2018/02/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6.png)

unset($a)之后，减少了一次引用的recount，这个时候，已经没有了外部的引用，但是还有一个内部还有一个元素指向该引用。

![](/uploads/2018/02/%E6%9C%AA%E5%91%BD%E5%90%8D%E6%96%87%E4%BB%B6-2.png)

像这种因为循环指向没办法释放的变量称之为垃圾。PHP引入了另外的一种机制来进行垃圾回收。

*   如果一个变量的value的refcount减少到0，说明这个value可以释放，那么这就不属于垃圾
*   如果一个变量的value减少之后大于0，那么这个value还不能被释放，那么这个value就是垃圾。 所以，判断一个变量是不是垃圾，要看value的refcount是否减少到了0。

目前垃圾回收只会出现在array和object两种类型中，当一个value被视为垃圾的时候，PHP会将这个value收集起来，等到打到了规定的数量，启动垃圾回收机制，进行统一的释放。

#### 回收的时机

前面说了，PHP垃圾回收并不是产生一个垃圾value，就进行释放，而是把value收集起来统一释放，以为value的分析和释放，也是有性能消耗的。 在php.ini中，`zend.enable_gc`用来设置是否启动垃圾回收机制。绝大多数都是默认开启的，因为每个都有可能在写程序的时候，出现内存垃圾，如果把这个配置关闭了，那么就有可能造成所谓的垃圾泄露。 除了`zend.enable_gc`以为，还会配合`zend/zend_gc.c`里面的变量`GC_ROOT_BUFFER_MAX_ENTRIES`实现垃圾回收，默认`GC_ROOT_BUFFER_MAX_ENTRIES`的值是10001，GC\_ROOT\_BUFFER\_MAX\_ENTRIES\[0\]是用来保存一些header的数据，GC\_ROOT\_BUFFER\_MAX\_ENTRIES\[1\]~GC\_ROOT\_BUFFER\_MAX\_ENTRIES\[10000\]用来收集垃圾的数据。如果你想强制执行垃圾回收，也可使用函数[gc\_collect\_cycles()](http://php.net/manual/zh/function.gc-collect-cycles.php)实现。

#### 参考文献

*   PHP7内核剖析
*   PHP手册