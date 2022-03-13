---
title: nohup后面2>&1的理解
tags:
  - Linux
id: '701'
categories:
  - - Linux
date: 2019-11-03 18:39:50
---

我们在后台运行命令的时候，除了会借助一些后台进程守护工具，也会用到Linux的nohup，比如：`nohup command > /dev/null 2>&1 &`。对于命令的含义，其实大家都知道，无外乎就是`不输出任何的错误信息`。但是对于技术，我更希望自己能够知其然而知其所以然

<!--more-->

### 1和2？

首先，先说下数字1和2的含义： `Linux shell中有三种输入输出，分别为标准输入，标准输出，错误输出，分别对应0，1，2`

### 分解命令

其实命令`nohup command > /dev/null 2>&1 &` 应该进行拆分为多个部分

*   nohup
*   command > /dev/null 2>&1
*   &

第一部分肯定不用说，就是nohup的用法，第三部分的含义，就是代码后台运行命令，其实也不用说。令人费解的就是第二部分的含义。

开头的时候，我有说过，其实数字是Linux规定的一种输出的信号，在编程语言中，&通常是代表变量的内存地址，所以 2>&1代表了，把`错误输出`写到`标准输出`里面。

### 举个例子

```bash
$ ll
total 72
-rw-r--r--  1 sf  staff     36 10 30 11:17 1
-rw-r--r--  1 sf  staff    111 10 30 10:40 CMakeLists.txt
drwxr-xr-x  7 sf  staff    224 10 30 10:41 cmake-build-debug
-rw-r--r--  1 sf  staff     96 10 30 10:40 main.cpp
-rw-------  1 sf  staff      0 10 30 11:11 nohup.out
-rwxr-xr-x  1 sf  staff  12732 10 30 10:45 sigtest
-rw-r--r--  1 sf  staff    258 10 30 10:45 sigtest.cpp
-rw-r--r--  1 sf  staff     23 10 30 11:02 tt.txt
```

#### 验证数字1是不是标准输出的意思

```bash
# shell-demo0
$ ll ./ 1>output.txt
$ cat output.txt
total 64
-rw-r--r--  1 sf  staff    111 10 30 10:40 CMakeLists.txt
drwxr-xr-x  7 sf  staff    224 10 30 10:41 cmake-build-debug
-rw-r--r--  1 sf  staff     96 10 30 10:40 main.cpp
-rw-------  1 sf  staff      0 10 30 11:11 nohup.out
-rw-r--r--  1 sf  staff      0 10 30 11:48 output.txt
-rwxr-xr-x  1 sf  staff  12732 10 30 10:45 sigtest
-rw-r--r--  1 sf  staff    258 10 30 10:45 sigtest.cpp
-rw-r--r--  1 sf  staff     23 10 30 11:02 tt.txt
```

```bash
# shell-demo1
$ ll mmm 1>output1.txt
ls: mmm: No such file or directory
$ cat output1.txt
$
```

上面的两个例子证明了一个问题

*   shell-demo0在output.txt文件里面的内容，和ll的显示内容一致
*   shell-demo1中，我查看了一个不存在的文件，但是错误信息没有写入到文件output1.txt，而是输出到了屏幕上

结论：数字1确实是标准输出的含义

#### 验证数字2是不是错误信息的含义

```bash
#shell-demo2
$ ll ./ 2>output2.txt
total 72
-rw-r--r--  1 sf  staff    111 10 30 10:40 CMakeLists.txt
drwxr-xr-x  7 sf  staff    224 10 30 10:41 cmake-build-debug
-rw-r--r--  1 sf  staff     96 10 30 10:40 main.cpp
-rw-------  1 sf  staff      0 10 30 11:11 nohup.out
-rw-r--r--  1 sf  staff    443 10 30 11:48 output.txt
-rw-r--r--  1 sf  staff      0 10 30 11:51 output1.txt
-rw-r--r--  1 sf  staff      0 10 30 11:56 output2.txt
-rwxr-xr-x  1 sf  staff  12732 10 30 10:45 sigtest
-rw-r--r--  1 sf  staff    258 10 30 10:45 sigtest.cpp
-rw-r--r--  1 sf  staff     23 10 30 11:02 tt.txt
$ cat output2.txt
$ 
```

```bash
#shell-demo3
$ ll mmm 2>output3.txt
$ cat output3.txt
ls: mmm: No such file or directory
```

上面的两个例子证明了一个问题

*   shell-demo2在使用数字2的时候，当前目录下的文件信息在屏幕输出，没有写入到文件output2.txt
*   shell-demo3中，错误信息输出到了output3.txt

结论：数字2确实是错误输出的含义

#### 2>1和2>&1

接下来，我们对比下2>1和2>&1的区别

```bash
#shell-demo4
$ ll ./ mm >output4.txt 2>1
$ cat output4.txt
./:
total 72
-rw-r--r--  1 sf  staff     34 10 30 13:33 1
-rw-r--r--  1 sf  staff    111 10 30 10:40 CMakeLists.txt
drwxr-xr-x  7 sf  staff    224 10 30 10:41 cmake-build-debug
-rw-r--r--  1 sf  staff     96 10 30 10:40 main.cpp
-rw-------  1 sf  staff      0 10 30 11:11 nohup.out
-rw-r--r--  1 sf  staff      0 10 30 13:33 output4.txt
-rwxr-xr-x  1 sf  staff  12732 10 30 10:45 sigtest
-rw-r--r--  1 sf  staff    258 10 30 10:45 sigtest.cpp
-rw-r--r--  1 sf  staff     23 10 30 11:02 tt.txt
```

```bash
#shell-demo5
$ ll ./ mm >output5.txt 2>&1
$ cat output5.txt
ls: mm: No such file or directory
./:
total 88
-rw-r--r--  1 sf  staff     34 10 30 13:33 1
-rw-r--r--  1 sf  staff    111 10 30 10:40 CMakeLists.txt
drwxr-xr-x  7 sf  staff    224 10 30 10:41 cmake-build-debug
-rw-r--r--  1 sf  staff     96 10 30 10:40 main.cpp
-rw-------  1 sf  staff      0 10 30 11:11 nohup.out
-rw-r--r--  1 sf  staff    527 10 30 13:34 output4.txt
-rw-r--r--  1 sf  staff     34 10 30 13:34 output5.txt
-rwxr-xr-x  1 sf  staff  12732 10 30 10:45 sigtest
-rw-r--r--  1 sf  staff    258 10 30 10:45 sigtest.cpp
-rw-r--r--  1 sf  staff     23 10 30 11:02 tt.txt
```

对比应该比较明显了吧。

`如果是使用2>1，那么代表错误信息会写入到标准信息里面，但是会被标准信息所覆盖`

`如果是使用2>&1，那么代表错误信息会写入到标准信息的内存地址里，然后再和标准信息一起写到输出文件里面`

本文链接：[https://feilong.tech/2019/11/03/nohup/](https://feilong.tech/2019/11/03/nohup/)