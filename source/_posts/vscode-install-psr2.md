---
title: vscode安装psr2格式化工具
tags: []
id: '288'
categories:
  - - PHP
comments: false
date: 2018-07-26 00:14:21
---

### 关于

PSR应该算得上是PHP比较权威的codestyle的标准了，大多数的PHP届都是沿用psr的风格。当然，其中也会或多或少的有一些自己的风格，比如PHP Ci框架，也有一些自己的代码风格。

vscode是我用了这么多编译器中最好用的了，它虽然不想sublime那样轻便，但是他自带的一些工具非常好用。比如，集成git之后，不用再使用命令行提交变更，解决冲突直接一个按钮操作。用金星姐姐的话来说，“完美”。

<!--more-->

### 开始安装

VScode有一款插件，是用来做PHP代码格式化的，叫做 `PHP Formatter`， 如果你们公司使用PSR的命名规范，那么这个简直就是福音啊。 下载链接 [https://marketplace.visualstudio.com/items?itemName=Sophisticode.php-formatter](https://marketplace.visualstudio.com/items?itemName=Sophisticode.php-formatter)

#### 安装php-cs-fixer

```bash
$ composer require fabpot/php-cs-fixer #使用composer安装依赖
```

![](/uploads/2018/07/QQ20180725-235749@2x-1024x350.png)

#### 在VScode上面下载插件

![](/uploads/2018/07/QQ20180726-000004-1024x859.png)

#### 配置插件

VScode->Code->首选项->设置 在`用户设置`增加截图的json的配置项 ![](/uploads/2018/07/475A8C81-7869-4446-AD83-3EC62F51A6E7-1024x588.png)

### enjoin