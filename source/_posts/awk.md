---
title: awk命令的简单介绍
tags:
  - Linux
id: '271'
categories:
  - - Linux
comments: false
date: 2018-04-08 23:54:50
---

#### 背景

awk算是Linux上面比较实用频繁的命令之一。第一次见到这个命令，是同事们分析一些日志实用，通过这个命令与其他命令结合，可以有效的分析nginx日志的一些访问情况。所以我也特意找了一些资料，查询了一下。

#### 语法规则

awk的命令的语法规则是 `awk '条件类型1{动作1} 条件类型2{动作2} ...' 文件名；` 。awk条件类型后面的{}是满足条件后处理的一些动作。这些动作可以形成一套连续的操作。awk的处理单元是每一行。也就是每行处理之后，再对下一行进行处理。所以，awk并不适合对大量数据处理。

<!--more-->


#### awk的处理原理

```bash
feilongdeMBP:~ feilong$ awk '{print $0}' /etc/passwd
##
# User Database
#
# Note that this file is consulted directly only when the system is running
# in single-user mode.  At other times this information is provided by
# Open Directory.
#
# See the opendirectoryd(8) man page for additional information about
# Open Directory.
##
nobody:*:-2:-2:Unprivileged User:/var/empty:/usr/bin/false
root:*:0:0:System Administrator:/var/root:/bin/sh
daemon:*:1:1:System Services:/var/root:/usr/bin/false
_uucp:*:4:4:Unix to Unix Copy Protocol:/var/spool/uucp:/usr/sbin/uucico
_taskgated:*:13:13:Task Gate Daemon:/var/empty:/usr/bin/false
_networkd:*:24:24:Network Services:/var/networkd:/usr/bin/false
_installassistant:*:25:25:Install Assistant:/var/empty:/usr/bin/false
_lp:*:26:26:Printing Services:/var/spool/cups:/usr/bin/false
.....
```

我们发现，这样输出的内容和执行`cat /etc/passwd`内容是一样的。

```bash
feilongdeMBP:~ feilong$ awk '{print $1}' /etc/passwd
##
#
#
#
#
#
#
#
#
##
nobody:*:-2:-2:Unprivileged
root:*:0:0:System
daemon:*:1:1:System
_uucp:*:4:4:Unix
_taskgated:*:13:13:Task
_networkd:*:24:24:Network
_installassistant:*:25:25:Install
_lp:*:26:26:Printing
_postfix:*:27:27:Postfix
_scsd:*:31:31:Service
_ces:*:32:32:Certificate
_mcxalr:*:54:54:MCX
_appleevents:*:55:55:AppleEvents
_geod:*:56:56:Geo
_serialnumberd:*:58:58:Serial
_devdocs:*:59:59:Developer
_sandbox:*:60:60:Seatbelt:/var/empty:/usr/bin/false
_mdnsresponder:*:65:65:mDNSResponder:/var/empty:/usr/bin/false
.....
```

随着print后面的变化，输出的内容也发生了变化

所以，awk的原理是这样的

![](/uploads/2018/04/WX20180408-225116.png)

除了这个，awk还有一些标量的含义

标量

含义

NR

当前的行号

NF

每一行拥有的字段总数

FS

每行的字段分隔符（默认空格）

RS

每行的结束符（默认\\n）

#### 实际操作

##### 以分号进行分割

```bash
feilongdeMBP:~ feilong$ awk 'FS=":" {print $1}' /etc/passwd ## 或 awk -F ":" '{print $1}' /etc/passwd
##
# User Database
#
# Note that this file is consulted directly only when the system is running
# in single-user mode.  At other times this information is provided by
# Open Directory.
#
# See the opendirectoryd(8) man page for additional information about
# Open Directory.
##
nobody
root
daemon
_uucp
_taskgated
_networkd
_installassistant
_lp
_postfix
_scsd
_ces
_mcxalr
....
```

##### 比如，只看 第20行到30行的内容

```shell
feilongdeMBP:~ feilong$ awk '{if(NR>=20 && NR<=30) {print "行号是: " NR " " $0}}' /etc/passwd
行号是: 20 _scsd:*:31:31:Service Configuration Service:/var/empty:/usr/bin/false
行号是: 21 _ces:*:32:32:Certificate Enrollment Service:/var/empty:/usr/bin/false
行号是: 22 _mcxalr:*:54:54:MCX AppLaunch:/var/empty:/usr/bin/false
行号是: 23 _appleevents:*:55:55:AppleEvents Daemon:/var/empty:/usr/bin/false
行号是: 24 _geod:*:56:56:Geo Services Daemon:/var/db/geod:/usr/bin/false
行号是: 25 _serialnumberd:*:58:58:Serial Number Daemon:/var/empty:/usr/bin/false
行号是: 26 _devdocs:*:59:59:Developer Documentation:/var/empty:/usr/bin/false
行号是: 27 _sandbox:*:60:60:Seatbelt:/var/empty:/usr/bin/false
行号是: 28 _mdnsresponder:*:65:65:mDNSResponder:/var/empty:/usr/bin/false
行号是: 29 _ard:*:67:67:Apple Remote Desktop:/var/empty:/usr/bin/false
行号是: 30 _www:*:70:70:World Wide Web Server:/Library/WebServer:/usr/bin/false
```

##### 已知 test.txt 内容是 "I am Poe,my qq is 33794712"。过滤相应字符串，是输出结果为 "Poe 33794712"

```bash
feilongdeMBP:~ feilong$ awk -F "[ ,]+" '{print $3 " " $7}' test.txt
Poe 33794712
```

#### BEGIN和END模块

begin和end主要是只在awk执行开始（还没对第一行进行操作）和结束（对最后一行处理结束）后的行为。所以，begin和end只会操作一次。所以begin和end更像是 编程语言中的默认构造函数和析构函数。

##### 统计用户的数量

```bash
feilongdeMBP:Downloads feilong$ awk 'BEGIN{count = 0} {if (NR > 10) { count ++} } { if (NR > 10 ) { print $1}} END{print "总的用户数量是: " count}' /etc/passwd ## 以为我的机器上面前10行不是用户的数据
nobody:*:-2:-2:Unprivileged
root:*:0:0:System
daemon:*:1:1:System
_uucp:*:4:4:Unix
_taskgated:*:13:13:Task
_networkd:*:24:24:Network
_installassistant:*:25:25:Install
_lp:*:26:26:Printing
_postfix:*:27:27:Postfix
_scsd:*:31:31:Service
_ces:*:32:32:Certificate
_mcxalr:*:54:54:MCX
_appleevents:*:55:55:AppleEvents
_geod:*:56:56:Geo
_serialnumberd:*:58:58:Serial
_devdocs:*:59:59:Developer
_sandbox:*:60:60:Seatbelt:/var/empty:/usr/bin/false
_mdnsresponder:*:65:65:mDNSResponder:/var/empty:/usr/bin/false
_ard:*:67:67:Apple
_www:*:70:70:World
_eppc:*:71:71:Apple
总的用户数量是: 93
```

##### 总计金额

```shell
feilongdeMBP:~ feilong$ cat test.txt
Name    1st 2st 3st
Tyler   100 200 500
Start   59  30  444
Jack    345 222 67

feilongdeMBP:~ feilong$  awk 'BEGIN{ totle = 0;} NR==1{print "Name\t1st\t2st\t3st\tTotle"} NR>=2{totle = $2 + $3 + $4; print $1 "\t" $2 "\t" $3 "\t" $4 "\t"  totle}' test.txt
Name    1st 2st 3st Totle
Tyler   100 200 500 800
Start   59  30  444 533
Jack    345 222 67  634
```

##### 统计字节数量

```bash
feilongdeMBP:~ feilong$ ll  grep JPG
-rw-r--r--@   1 feilong  access_bpf    255800  3  1 16:49 IMG_0898.JPG
-rw-r--r--@   1 feilong  access_bpf    258234  3  1 16:49 IMG_0899.JPG
-rw-r--r--@   1 feilong  access_bpf    338363  3  4 10:32 IMG_0930.JPG

feilongdeMBP:~ feilong$ ll  grep JPG  awk 'BEGIN{size = 0;} {size += $5} END{print "The .JPG file size:" size/1024/1024 "MB"}'
The .JPG file size:0.812909MB
```

#### awk还有丰富的运算符

awk支持大多数的运算符，这些运算符和编程语言基本类似

![](/uploads/2018/04/1089507-20170126224150269-207487187.jpg)

#### 正则表达式

语法结构 `awk '/正则表达式/{动作}' 文件`

##### 找出匹配包含root的行

```shell
feilongdeMBP:~ feilong$ awk '/root/{print $0}' /etc/passwd
root:*:0:0:System Administrator:/var/root:/bin/sh
daemon:*:1:1:System Services:/var/root:/usr/bin/false
_cvmsroot:*:212:212:CVMS Root:/var/empty:/usr/bin/false
```

#### 其他

awk还有其他的功能，比如支持for循环，if语句，while循环等待

##### for 循环

```bash
feilongdeMBP:~ feilong$ awk '/root/{print $0; for(i=1; i< 4; i++) {print "test"}}' /etc/passwd
root:*:0:0:System Administrator:/var/root:/bin/sh
test
test
test
daemon:*:1:1:System Services:/var/root:/usr/bin/false
test
test
test
_cvmsroot:*:212:212:CVMS Root:/var/empty:/usr/bin/false
test
test
test
```

#### 参考资料

*   [Linux三剑客之awk命令](https://www.cnblogs.com/ginvip/p/6352157.html)
*   鸟哥的Linux私房菜基础学习篇