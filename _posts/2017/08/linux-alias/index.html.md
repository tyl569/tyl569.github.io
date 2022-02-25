---
layout: post
title: Linux alias 永久生效
date: 2017-08-24
tags: ["Linux"]
---

#### alias 是用来设置命令别名的，使用这个命令，可以极大的节省我们的时间

#### 比如设置 `alias db='mysql -uroot -proot'`,设置之后，可以直接使用db命令登录mysql了，但是当关闭控制台的时候，alias就会失效。

<!--more-->

#### alias永久生效

#### 在`/home`用户目录下面有个隐藏文件_`.bashrc`_, 使用vim打开，然后在文档后面追加alias命令即可

    $ vim /home/ubuntu/.bashrc

    ##在结尾追加alias命令，如alias db = 'mysql -uroot -proot'

    $ source ~/.bashrc ##使alias生效
    