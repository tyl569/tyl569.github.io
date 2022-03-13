---
title: 将MySQL数据导出到Excel
tags: []
id: '294'
categories:
  - - Mysql
date: 2018-08-07 12:23:18
---

#### 背景

有一天PM跑来找我，让我导出一部分数据。之前做导出的功能，基本上都是依靠代码的方式，如果针对这种临时性需求，如果还要临时写代码，加上自测，一天的时间就这样没了。

<!--more-->

#### 执行

```bash
$ mysql -uroot -proot -e 'select * from test' > test.xls
```

#### 发现问题

本来以为只要这条命令就行，但是发现导出的Excel是乱码，原来是因为配置文件里面，客户端连接的编码方式不是utf8

#### 重试

```bash
$ mysql -uroot -proot -e 'set names utf8; select * from test' > test.xls
```

网上找了一下方案，需要使用记事本打开，然后另存为中文的编码方式。

但是我使用Mac打开的时候，没有找到`另存为`选项 - -

![](/uploads/2018/08/WX20180807-121223@2x.png)

#### 解决办法

文件->复制->存储(副本)->纯文本编码:(中文 GB18030)->更改文件拓展名为xls 即可

![](/uploads/2018/08/20180807121627.jpg)

![](/uploads/2018/08/WX20180807-122030@2x.png)