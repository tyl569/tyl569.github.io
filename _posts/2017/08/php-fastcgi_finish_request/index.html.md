---
layout: post
title: PHP使用fastcgi_finish_request 提高响应
date: 2017-08-24
tags: ["Linux"]
---

#### 当PHP在运行fastcgi模式的时候，php-fpm提供了一个函数，叫做`fastcgi_finish_request`的函数，来加快处理请求，如果有些后端请求可以先生成页面再处理的话，可以选择这个函数。

<!--more-->

#### 听起来挺迷茫的，下面给大家举个例子

    <?php
        echo 'This is example1';
        fastcgi_finish_request();
        echo 'This is example2';
        file_put_contents('/var/log/test.log', 'hello world');

    ?>

#### 当通过浏览器访问的时候，页面会输出`This is example1`，但是没有输出`This is example2`同时，也把内容写入到了日志文件中。这说明，当PHP执行到了fastcgi_finish_request的时候，服务器就和客户端结束了请求，但是服务器还是继续"异步"执行。

#### 再来个直观的例子

    <?php
        echo 'This is example1';
        fastcgi_finish_request();
        echo 'This is example2';
        file_put_contents('/var/log/test.log', time('Y-m-d H:i:s', time()), FILE_APPEND);
        sleep(1);
        file_put_contents('/var/log/test.log', time('Y-m-d H:i:s', time()), FILE_APPEND);
        sleep(1);
        file_put_contents('/var/log/test.log', time('Y-m-d H:i:s', time()), FILE_APPEND);
    ?>

#### 执行的结果就是页面值输出了`This is example1`,服务器记录了三条日志信息。

#### 个人觉得这个函数很有用，尤其是在客户端与云端进行交互的时候，可以很快缩短响应时间，把多余的请求交给服务器来处理。

#### 另外，在代码移植上建议加上如下代码

    <?php
        if (!function_exists("fastcgi_finish_request")) {
              function fastcgi_finish_request()  {
              }
        }
    ?>

#### fastcgi_finish_request虽然很好用，但是也有很多限制：

*   PHP fastcgi有进程限制，如果并发太大的话，没办法处理新的请求，nginx可能会出现502 bad request

内容参考鸟哥博客[使用fastcgi_finish_request提高页面响应速度](http://www.laruence.com/2011/04/13/1991.html)