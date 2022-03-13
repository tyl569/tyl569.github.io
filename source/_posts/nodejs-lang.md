---
title: NodeJs 多语言包
tags: []
id: '127'
categories:
  - - Nodejs
date: 2017-08-24 20:03:23
---

公司的项目是PHP CI ( CodeIgniter ) 框架，框架有很多功能，可以切换多种语言包，Chinese/English。今天接到新需求，原有的Nodejs项目，也要实现多语言问题，在网上找到好多资料，也没有找到有用的。后来就在[Github](https://github.com)找了找资源，废了九牛二虎之力终于找到了。
<!-- more -->
#### 1) 安装语言包langs

命令和安装express差不多：

```bash
$ npm install langs --save
```

#### 2) 引入拓展

```bash
var langs = require('langs');
```

#### 3) 获取url参数，根据参数判断语言类别

```bash
exports.index = function (req, res, next) { 
    var query = req.query;
    var lang = query.lang;
    var user_id = query.user_id;
    var token = query.token;
    if (lang == 'chinese') {
        var lang = langs.where("name", "Chinese");
    } else {
        var lang = langs.where("name", "English");
    }
    res.render('index', {json_data});
}
```

#### 4) 具体语言配置在 node\_modules/langs/data.json 中配置

#### 5) 拓展位置[nodejs-langs](https://github.com/adlawson/nodejs-langs)