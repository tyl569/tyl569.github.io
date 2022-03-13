---
title: Linux自定义PHP的环境变量
tags:
  - Linux
  - nginx
  - PHP
id: '254'
categories:
  - - Linux
  - - Nginx
  - - PHP
comments: false
date: 2018-03-07 17:41:46
---

很多时候，我们会使用 PHP的`$_SERVER`数组，通过这个数组，可以获取一些服务器的变量信息。但是不同的模式下，这个全局数组是不一样的。比如，在web模式下，`$_SERVER` 是获取的fastcgi\_params，在cli模式下，获取的是环境变量(也就是常见的Linux 的export设置的)

<!--more-->

举个例子，我们要设置$\_SERVER\['AAAAA'\]='test\_data'

刚开始，不管web模式下，还是cli模式下，都是没有这个值的。

web模式 ![](/uploads/2018/03/WX20180307-173333-300x55.png)

cli模式 ![](/uploads/2018/03/WX20180307-173521.png)

#### 更改nginx 的环境变量

找到fastcgi\_params文件，一般是和nginx.conf在同一个目录，

![](/uploads/2018/03/WX20180307-173138.png)

```bash
$ sudo nginx -s reload 
```

然后刷新页面 ![](/uploads/2018/03/WX20180307-173727.png)

#### 更改cli模式先的环境变量

```bash
$ vim ~/.bashrc
```

![](/uploads/2018/03/WX20180307-173048.png)

```bash
$ source ~/.bashrc
$ php -r 'var_dump($_SERVER["AAAAA"]);';
```

![](/uploads/2018/03/WX20180307-173922.png)