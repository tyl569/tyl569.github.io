---
layout: post
title: PHP 7 拓展编写--从hello world开始
date: 2017-08-24
tags: ["Linux","PHP"]
---

本文主要以PHP7为基础，学习从0开始编写PHP7 拓展，拓展的主要功能是通过拓展，实现如下代码的效果

    <?php
    function say() {
        return 'hello world';
    }
    echo say(); // hello world
    ?>

<!--more-->

##### 在拓展中实现say函数，输出hello world

#### 第一步：生成代码

##### PHP给我们提供了拓展的生成工具，`ext_skel`。这个工具在PHP源码的 `ext/`文件夹下面，而且在ext/文件夹下是各种拓展目录。

    $ cd php_src/ext/
    $ ext_skel --extname=say

##### `--extname` 参数是设置拓展的名字。执行完这么命令后，`ext`文件夹下就会生成一个say的文件夹

#### 第二步：修改config.m4配置文件

##### config.m4 的作用是为了配合phpize工具生成configure文件。configure文件是为了用于环境监测，监测编译的环境是否满足条件。现在我们开始修改config.m4配置文件

    $ cd say/
    $ vim config.m4

##### 打开config.m4文件，会有看到文件开头有如下内容

    dnl If your extension references something external, use with:

    dnl PHP_ARG_WITH(say, for say support,
    dnl Make sure that the comment is aligned:
    dnl [  --with-say             Include say support])

    dnl Otherwise use enable:

    dnl PHP_ARG_ENABLE(say, whether to enable say support,
    dnl Make sure that the comment is aligned:
    dnl [  --enable-say           Enable say support])

##### 其中dnl是注释，上面的内容大概是说：如果你编写的拓展依赖其他拓展，就使用去掉PHP_ARG_WITH注释内容，否则去掉PHP_ARG_ENABLE注释内容。我们这里不需要依赖其他拓展，所以我们去掉PHP_ARG_ENABLE注释内容。去掉后的内容如下：

    dnl If your extension references something external, use with:

    dnl PHP_ARG_WITH(say, for say support,
    dnl Make sure that the comment is aligned:
    dnl [  --with-say             Include say support])

    dnl Otherwise use enable:

    PHP_ARG_ENABLE(say, whether to enable say support,
    Make sure that the comment is aligned:
    [  --enable-say           Enable say support])

#### 第三步：代码实现

##### 修改say.c代码 ，实现say方法。找到 PHP_FUNCTION(confirm_say_compiled) 方法，在上面增加如下方法：

    PHP_FUNCTION(say)
    {
        zend_string *strg;
        strg = strpprintf(0, "hello world");
        RETURN_STR(strg);
    }

##### 然后找到PHP_FE(confirm_say_compiled,    NULL)在上面增加PHP_FE(say, NULL) ，如果不加上这个，那么say函数将不能引用，加完后的代码如下：

    const zend_function_entry say_functions[] = {
            PHP_FE(say, NULL)                       /* For testing, remove later. */
            PHP_FE(confirm_say_compiled,    NULL)           /* For testing, remove later. */
            PHP_FE_END      /* Must be the last line in say_functions[] */
    };

其实在zend_function_entry 上面，你会看到`Every user visible function must have an entry in say_functions[]`，意思是说每个用户的函数必须定义在say_functions函数里面，否则在PHP使用say函数的时候报出函数未定义的错误`Fatal error: Uncaught Error: Call to undefined function say()`

#### 第四步：编译安装

    $ phpize
    $ ./configure
    $ make && make install

##### 然后修改php.ini，在配置文件结尾加上如下代码

可以使用`php --ini`查看配置文件的位置

    [say]
    extsion=say.so

##### 然后执行php -m 查看是否已经把say函数加载进来：

    [PHP Modules]
    Core
    ctype
    date
    dom
    fileinfo
    filter
    gd
    hash
    iconv
    json
    libxml
    pcre
    PDO
    pdo_sqlite
    Phar
    posix
    Reflection
    say
    session
    SimpleXML
    SPL
    sqlite3
    standard
    tokenizer
    xml
    xmlreader
    xmlwriter

    [Zend Modules]

#### 第五步：运行测试

##### 编写PHP脚本，进行测试

    <?php
    echo say();
    ?>

##### 结果输出：

    $ php test.php
    $ hello world