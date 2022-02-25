---
layout: post
title: Linux 将用户加入用户组
date: 2017-08-24
tags: ["Linux"]
---

#### 准备工作

两个账户：tengyunlong test
如果没有账户使用 adduser进行创建

    $ adduser test
    $ usermod -a -G test test # 将test用户加入到test用户组
    $ groups test #查看test的所有分组
    $ test : test 

<!--more-->

使用test用户在/home/test/路径下新建test.txt使此文件允许test分组和test用户读写

    total 36
    drwxr-xr-x 2 test test 4096 Jun  1 15:51 ./
    drwxr-xr-x 4 root root 4096 Jun  1 15:49 ../
    -rw-r--r-- 1 test test  220 Jun  1 15:49 .bash_logout
    -rw-r--r-- 1 test test 3637 Jun  1 15:49 .bashrc
    -rw-r--r-- 1 test test 8980 Jun  1 15:49 examples.desktop
    -rw-r--r-- 1 test test  675 Jun  1 15:49 .profile
    -rw-rw-r-- 1 test test    8 Jun  1 15:53 test.txt

#### 添加分组

使用root账户将 tengyunlong加到test分组

    $ usermod -a -G test tengyunlong
    $ groups tengyunlong
    tengyunlong : tengyunlong adm cdrom sudo dip plugdev lpadmin sambashare test #可以看到tengyunlong加到test分组了

#### 运行测试

切换tengyunlong账户，进入/home/test目录下

    $ ll
    total 36
    drwxr-xr-x 2 test test 4096 Jun  1 15:51 ./
    drwxr-xr-x 4 root root 4096 Jun  1 15:49 ../
    -rw-r--r-- 1 test test  220 Jun  1 15:49 .bash_logout
    -rw-r--r-- 1 test test 3637 Jun  1 15:49 .bashrc
    -rw-r--r-- 1 test test 8980 Jun  1 15:49 examples.desktop
    -rw-r--r-- 1 test test  675 Jun  1 15:49 .profile
    -rw-rw-r-- 1 test test    8 Jun  1 15:53 test.txt
    $  echo 'sssssssssss'> test.txt
    $ cat test.txt 
    sssssssssss #可以看到内容被写入了