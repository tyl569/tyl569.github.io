---
layout: post
title: Nodejs 使用pm2实现开机自启
date: 2017-08-24
tags: ["Linux","Nodejs"]
---

公司有个nodejs的云服务，但是没在开机自启的进程中，如果服务器因为某种原因 `reboot` 的话，服务就挂掉了。这肯定是不允许的。so 想要写个脚本，来实现开机自启。奈何 `shell` 太渣渣，搞不定。所以在社区找到了`pm2`，可以把`nodejs`加到自启服务中。

<!--more-->

#### pm2有一些优势:

> *   自带负载均衡功能的node应用进程管理器
> *   可以监控应用CPU和内存情况
> *   操作简单
> *   非常适合IaaS结构

#### pm2也有劣势:

> *   不适合PaaS结构

#### 拓展:

> *   SaaS: Software-as-a-Service 软件即服务，例如Google的Gmail，把软件做成服务
> *   IaaS: Infrastructure-as-a-Service 基础设施即服务，这是我们最常见的云端接口，网站等
> *   PaaS: Platform-as-a-Service 平台即服务，专门做平台服务，例如新浪云等
> 详细了解参见[云服务模式：SaaS、PaaS和IaaS，哪一种适合你？](http://cloud.51cto.com/art/201107/278903.htm)

#### 1、全局安装pm2

    $ npm install pm2 -g

#### 2、找到项目的目录，并使用pm2启动node服务

    $ cd /usr/share/nginx/wechat-iot
    $ pm2 start app.js
    [PM2] Starting app.js in fork_mode (1 instance)
    [PM2] Done.
    ┌──────────┬────┬──────┬───────┬────────┬─────────┬────────┬─────────────┬──────────┐
    │ App name │ id │ mode │ pid   │ status │ restart │ uptime │ memory      │ watching │
    ├──────────┼────┼──────┼───────┼────────┼─────────┼────────┼─────────────┼──────────┤
    │ app      │ 0  │ fork │ 12120 │ online │ 0       │ 0s     │ 15.863 MB   │ disabled │
    └──────────┴────┴──────┴───────┴────────┴─────────┴────────┴─────────────┴──────────┘
     Use `pm2 show <id'name>` to get more details about an app

#### 3、把node服务加到进程

    $ pm2 startup centos #pm2 startup ubuntu
    $ pm2 save 

#### 其他命令

    $ pm2 stop app.js #停止node服务
    $ pm2 restart app.js #重启node服务
    $ pm2 delete app.js #在进程中删除
    $ pm2 status #查看状态
    $ pm2 monit #查看占用的CPU和内存