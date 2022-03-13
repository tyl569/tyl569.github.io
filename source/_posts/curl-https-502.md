---
title: 一次由于curl调用https接口导致的502
tags: []
id: '251'
categories:
  - - Linux
  - - Nginx
  - - PHP
comments: false
date: 2018-03-06 20:49:06
---

#### 背景描述

最近在调研百度人脸识别的服务，百度的人脸识别是免费的，但是有QPS的限制，QPS免费的最大值是5，也就是峰值在每秒5次，都是可以免费使用的。我从百度平台下载的SDK，但是到了第一步就被卡主了。

当我使用检测接口的时候，频繁出现 502。在之前，本地运行PHP的时候，也会偶尔出现502，但是并没有这么高。这次变态的浮现率是100%.

查了下nginx日志`*173 upstream prematurely closed connection while reading response header from upstream, client: 127.0.0.1, server: sdeno-api, request: "GET /facev1/test HTTP/1.1", upstream: "fastcgi://127.0.0.1:9000"`

在网上查了一下，说什么的都有，大多数是在说nginx的问题。没有一个能够解决问题。

#### 发现问题

我发现，出现http 502 的接口使用了curl，所以很可能是curl代码有问题。我自己有写了一段代码，调用百度首页`http://www.baidu.com`，咦？正常返回啊。完全没有问题。后来百度查了很多，最终发现一个不起眼的问题 `php curl 调用https出现502问题`。我验证了一些，我曹，果然只有https接口是不正常的。

#### 解决办法

后来找到了一个靠谱的方法，那就是使用sudo重启php-fpm.

```bash
$ brew services stop php53
$ sudo brew services start php53
```

其实除了这样，还可以考虑下重装或者升级PHP的版本。我把本地的php5.3卸载了，重新安装了php5.6也解决了问题。

```bash
$ brew uninstall php53
$ brew install php56
```