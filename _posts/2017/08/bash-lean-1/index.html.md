---
layout: post
title: bash基础学习一
date: 2017-08-24
tags: ["Linux"]
---

### Bash 中处理特殊字符

### 符号

#### 注释

行号以#开头是注释，bash脚本的第一行通常是`#!/bin/bash`，意思是这个文件是bash脚本 `#!` 用于当前脚本的解释器

当然，在echo中转义的#是不能做转义的：

<!--more-->

    $ vim test.sh

输入如下代码，并保存

    #!/bin/bash

     echo "The # here does not begin a comment."
     echo 'The # here does not begin a comment.'
     echo The \# here does not begin a comment.
     echo The # 这里开始一个注释

     echo ${PATH#*:}         # 参数替换，不是一个注释
     echo $(( 2#101011 ))   # 数制转换（使用二进制表示），32+8+2+1

执行结果

    $ bash test.sh

运行效果

    The # here does not begin a comment.
    The # here does not begin a comment.
    The # here does not begin a comment.
    The
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
    43

### 分号

#### 命令分隔符

使用分号可以分割在同一行的两个或两个以上的命令

    $ vim test2.sh

输入如下代码，并保存

    #!/bin/bash
     echo hello; echo there
     filename=ttt.sh
     if [ -r "$filename" ]; then    # 注意: "if"和"then"需要分隔
         echo "File $filename exists."; cp $filename $filename.bak
     else
         echo "File $filename not found."; touch $filename
     fi; echo "File test complete."

运行脚本

    $ bash test2.sh

运行结果

    hello
    there
    File ttt.sh not found
    Filename test complete.

#### 终止case选项(双;分号)

使用双分号用于终止case选项

    $ vim test.bash

输入如下代码，并保存

    #!/bin/bash
    varname=b
    case $varname in 
        [a-z]) echo "abc";;
        [0-9]) echo "123";;
    esac    #case终止符号

执行脚本，查看输出

    $ bash test.sh
    abc

解释说明，上面的代码，首先赋值给变量varname的值是b，然后使用case进行判断。
case的格式

    case $ in
        条件1) command;;
        条件2) command;;
        .
        .
        .
        *) command;; ##匹配所有
    esac

### 点号（.）

等价于source命令
bash 中的 source 命令用于在当前 bash 环境下读取并执行 FileName.sh 中的命令。

    $ source test.sh
    Hello World
    $ . test.sh
    Hello World

### 引号

#### 双引号

"STRING" 将会阻止（解释）STRING中大部分特殊的字符。

#### 单引号

'STRING' 将会阻止STRING中所有特殊字符的解释，这是一种比使用"更强烈的形式。

### 反引号

反引号通常是用于命令，反引号中的命令会优先执行

    $ cp `mkdir back` test.sh back

分析：反引号类似算法中的小括号，会优先执行，上面的运行结果

    $ cp `mkdir back` test.sh back
    $ ll
    total 16K
    drwxrwxr-x 3 shiyanlou shiyanlou 4.0K Apr 14 14:43 Code
    drwxrwxr-x 2 shiyanlou shiyanlou 4.0K Nov 27  2015 Desktop
    drwxrwxr-x 2 shiyanlou shiyanlou 4.0K Apr 14 15:07 back
    -rw-rw-r-- 1 shiyanlou shiyanlou   13 Apr 14 15:04 test.sh

为了验证一下，我们可以删除back文件夹内容，然后执行 **cp test.sh back `mkdir back`** 效果是一样的

### 冒号

#### 空命令

等价于"NOP"（no op，一个什么也不干的命令）。也可以被认为与shell的内建命令true作用相同。":"命令是一个bash的内建命令，它的退出码（exit status）是（0）。
如：

    #!/bin/bash

    while :
    do
        echo "endless loop"
    done

等价于

    #!/bin/bash

    while true
    do
        echo "endless loop"
    done

可以在 if/then 中作占位符：

    condition=5

    if [ $condition -gt 0 ]
    then :   # 什么都不做，退出分支
    else
        echo "$condition"
    fi

#### 变量拓展/子串替换

在与`>`重定向操作符联合使用的时候，会把文件内容清空，但是并不会修改文件的权限。如果文件不存在，那么创建这个文件，在这方面，类似`cat`命令

    $ : > test.bash
    # 与 cat /dev/null > test.sh
    # /dev/null一直是一个空内容

在与`>>`重定向操作符结合使用时，将不会对预先存在的目标文件(: >> target_file)产生任何影响。如果这个文件之前并不存在，那么就创建它。
":"还用来在 /etc/passwd 和 $PATH 变量中做分隔符，如：

    $ echo $PATH
    /usr/local/bin:/bin:/usr/bin:/usr/X11R6/bin:/sbin:/usr/sbin:/usr/games

### 问号

#### 操作符

在一个双括号结构中，? 就是C语言的三元操作符，如：

    #!/bin/bash
     a=10
     (( t=a<50?8:9 ))
     echo $t