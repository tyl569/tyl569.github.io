---
layout: post
title: Ubuntu查看版本
date: 2017-08-24
tags: ["Linux"]
---

#### 使用命令：cat /proc/version 查看

    $ cat /proc/version
    Linux version 4.4.0-47-generic (buildd@lcy01-03) (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.2) ) #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016

<!--more-->

#### 使用命令：uname -a 查看

    $ uname -a
    Linux ip-172-31-9-166 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux

#### 使用命令：lsb_release -a 查看

    $ lsb_release -a
    Linux ip-172-31-9-166 4.4.0-47-generic #68-Ubuntu SMP Wed Oct 26 19:39:52 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
    ubuntu@ip-172-31-9-166:~/Download/php-5.6.28$ lsb_release -a
    No LSB modules are available.
    Distributor ID: Ubuntu
    Description:    Ubuntu 16.04.1 LTS
    Release:        16.04
    Codename:       xenial
    