---
title: 'Linux登录提示"-bash: ulimit: open files: 无法修改 limit 值: 不允许的操作"'
tags:
  - cli
  - Linux
  - shell
  - ulimit
id: '636'
categories:
  - - Linux
date: 2019-09-30 16:04:31
---

### 背景

自己在每次登录服务器的时候，都会出现 **\-bash: ulimit: open files: 无法修改 limit 值: 不允许的操作** 的提示信息，根据猜测，应该是登录的时候，执行了什么特殊的命令。通过百度查询了下，应该是登录的时候，执行了 ulimit 的命令。

### 原因猜测

一般登录的时候，都会调用 .bashrc 或者 .bash\_profile做一些初始化的操作。但是我找到对应的home下面的文件，并！没有！所以猜测，应该调用了全局的profile。

```bash
$ cat /etc/profile
....
ulimit -SHn 65535
```

果然最下面有一个ulimit的执行命令，把命令注释掉，重新登录，问题解决了

### 关于ulimit

ulimit的命令主要是用来控制shell程序的资源

```bash
$ help ulimit
ulimit: ulimit [-SHacdefilmnpqrstuvx] [限制]
    修改 shell 资源限制。

    在允许此类控制的系统上，提供对于 shell 及其创建的进程所可用的
    资源的控制。

    选项：
      -S    使用 `soft'（软）资源限制
      -H    使用 `hard'（硬）资源限制
      -a    所有当前限制都被报告
      -b    套接字缓存尺寸
      -c    创建的核文件的最大尺寸
      -d    一个进程的数据区的最大尺寸
      -e    最高的调度优先级（`nice'）
      -f    有 shell 及其子进程可以写的最大文件尺寸
      -i    最多的可以挂起的信号数
      -l    一个进程可以锁定的最大内存尺寸
      -m    最大的内存进驻尺寸
      -n    最多的打开的文件描述符个数
      -p    管道缓冲区尺寸
      -q    POSIX 信息队列的最大字节数
      -r    实时调度的最大优先级
      -s    最大栈尺寸
      -t    最大的CPU时间，以秒为单位
      -u    最大用户进程数
      -v    虚拟内存尺寸
      -x    最大的锁数量

    如果提供了 LIMIT 变量，则它为指定资源的新的值；特别的 LIMIT 值为
    `soft'、`hard'和`unlimited'，分别表示当前的软限制，硬限制和无限制。
    否则打印指定资源的当前限制值，不带选项则假定为 -f

    取值都是1024字节为单位，除了 -t 以秒为单位，-p 以512字节为单位，
    -u 以无范围的进程数量。

    退出状态：
    返回成功，除非使用了无效的选项或者错误发生。
```

通过ulimit，我们可以限制某个用户的使用的资源个数，比如我们限制用户打开文件的个数为2

```bash
$ ulimit -n 2
$ touch a.php
$ touch b.php
$ touch b.php
```

打开3个控制台，分别使用vim命令打开三个文件，当打开打开三个文件的时候，就出现了

```bash
$ vim c-bash: /dev/null: Too many open files
-bash: 重定向错误: 无法复制文件描述符: Invalid argument
-bash: 2: Invalid argument
-bash: /dev/null: Too many open files
-bash: 重定向错误: 无法复制文件描述符: Invalid argument
-bash: 2: Invalid argument
```