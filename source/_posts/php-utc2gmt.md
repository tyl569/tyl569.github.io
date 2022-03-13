---
title: PHP UTC转GMT时区
tags: []
id: '137'
categories:
  - - PHP
date: 2017-08-24 20:12:56
---

> 很长时间没有写blog了，并不是因为我偷懒了，而是最近没什么好写的东西，今天就费劲挤出一些东西，写一篇blog。
> 
> 背景：公司的项目海外市场比较多，所以需要兼容多时区问题。这块也是一个比较头疼的问题。

<!-- more -->

#### 用过ci的都知道，`index.php`的第一句话就是

```php
date_default_timezone_set('UTC');
```

#### 这就决定了，整个项目的时区是UTC时区。

#### 但是客户端传上来的时区基本上是GMT开头，例如`GMT+8(北京时间)`

#### 所以要把UTC时区转换成GMT时区

#### Code如下

```php
$time_zone = "GMT+8";
$time = time();
$date = date_create(date("Y-m-d H:i", $time), timezone_open('UTC'));
$date = date_timezone_set($date, timezone_open($time_zone));
$date = date_format($date, 'Y-m-d H:i');
```

#### 或者参照[php时区转换转换函数](http://www.poluoluo.com/jzxy/201401/258989.html):

```php
function toTimeZone($src, $from_tz = 'America/Denver', $to_tz = 'Asia/Shanghai', $fm = 'Y-m-d H:i:s') {
    $datetime = new DateTime($src, new DateTimeZone($from_tz));
    $datetime->setTimezone(new DateTimeZone($to_tz));
    return $datetime->format($fm);
}
```