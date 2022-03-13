---
title: Mac快捷安装PHP多版本
tags:
  - brew
  - Mac
  - PHP
id: '745'
categories:
  - - Linux
  - - PHP
date: 2020-12-13 19:30:41
---

## 程序员的苦恼

作为程序员，我们经常会面临一个比较痛苦的事情，那就是环境版本的问题。以PHP为例，有些框架或者工具，会对PHP版本有不同的要求。举个例子，我在公司开发使用的是PHP-7.3版本。但是周末在家，想做些其他有意思的事情，这个时候，发现有些框架或者工具的语言要求是>=PHP-7.0和<PHP-7.3。

有些同学会说，用docker啊！

没错，docker可以解决，但是有没有更加方便的工具来解决PHP版本切换的问题呢？当然有，那就是**phpbrew**!

<!--more-->

## phpbrew

项目地址，可以异步：[phpbrew](https://github.com/phpbrew/phpbrew/blob/master/README.cn.md "phpbrew")

phpbrew 主要解决了什么问题呢？

就像上面说的，它能更快和更加方便的让我们的Mac安装多个版本的PHP，以及PHP扩展，这样可以很快的提高我们的效率，作为Mac的PHP coder，也不用发愁找相应的PHP版本的解决方案。

## 使用方法

### 安装

```bash
$ curl -L -O https://github.com/phpbrew/phpbrew/releases/latest/download/phpbrew.phar
$ chmod +x phpbrew.phar

# Move the file to some directory within your $PATH
$ sudo mv phpbrew.phar /usr/local/bin/phpbrew
```

### 使用

初始化

```bash
phpbrew init
```

接着在 .bashrc 或 .zshrc 文件增加如下行：

```bash
[[ -e ~/.phpbrew/bashrc ]] && source ~/.phpbrew/bashrc
```

### 基本用法

列出已知的PHP版本

```bash
$ phpbrew known
Read local release list (last update: 2020-11-23 12:49:58 UTC).
You can run `phpbrew update` or `phpbrew known --update` to get a newer release list.
7.4: 7.4.12, 7.4.11, 7.4.10, 7.4.9, 7.4.8, 7.4.7, 7.4.6, 7.4.5 ...
7.3: 7.3.24, 7.3.23, 7.3.22, 7.3.21, 7.3.20, 7.3.19, 7.3.18, 7.3.17 ...
7.2: 7.2.34, 7.2.33, 7.2.32, 7.2.31, 7.2.30, 7.2.29, 7.2.28, 7.2.27 ...
7.1: 7.1.33, 7.1.32, 7.1.31, 7.1.30, 7.1.29, 7.1.28, 7.1.27, 7.1.26 ...
7.0: 7.0.33, 7.0.32, 7.0.31, 7.0.30, 7.0.29, 7.0.28, 7.0.27 ...
5.6: 5.6.40, 5.6.39, 5.6.38, 5.6.37, 5.6.36, 5.6.35, 5.6.34, 5.6.33 ...
5.5: 5.5.38, 5.5.37, 5.5.36, 5.5.35, 5.5.34, 5.5.33, 5.5.32, 5.5.31 ...
5.4: 5.4.45, 5.4.44, 5.4.43, 5.4.42, 5.4.41, 5.4.40, 5.4.39, 5.4.38 ...
```

### 安装拓展

```bash
$ phpbrew install 5.3.10 +mysql+sqlite+cgi

$ phpbrew install 5.3.10 +mysql+debug+pgsql +apxs2

$ phpbrew install 5.3.10 +pdo +mysql +pgsql +apxs2=/usr/bin/apxs2
```

### 查看安装的版本

```bash
$ phpbrew list
  php-7.2.34
* php-5.6.40
```

### 切换版本

```bash
$ phpbrew switch php-5.6.40
```

### 启动fpm

```bash
$ phpbrew fpm start
$ phpbrew fpm test
[13-Dec-2020 19:29:42] NOTICE: configuration file /Users/feilong/.phpbrew/php/php-5.6.40/etc/php-fpm.conf test is successful
```

### 查看版本

```bash
PHP 5.6.40 (cli) (built: Dec 13 2020 19:11:35)
Copyright (c) 1997-2016 The PHP Group
Zend Engine v2.6.0, Copyright (c) 1998-2016 Zend Technologies
```

## 遇到的问题

当我在安装php-7.2的时候，发生了一个问题，`checking for the location of zlib... configure: error: zip support requires ZLIB. Use --with-zlib-dir=<DIR> to specify prefix where ZLIB include and library are located`

这个是zlib的扩展没有找到对应的类库

如果没有安装，则先进行安装

```bash
$ brew reinstall zlib
==> Downloading https://homebrew.bintray.com/bottles/zlib-1.2.11.mojave.bottle.tar.gz
######################################################################## 100.0%
==> Reinstalling zlib
==> Pouring zlib-1.2.11.mojave.bottle.tar.gz
```

重新安装php

```bash
$  phpbrew install 7.2 -- \--with-zlib-dir=`brew --prefix zlib`
```

本文链接： [https://feilong.tech/2020/12/13/mac-phpbrew/](https://feilong.tech/2020/12/13/mac-phpbrew/)