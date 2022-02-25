---
layout: post
title: Nginx配置tcp代理
date: 2017-09-08
tags: ["Linux","Nginx","nginx","tcp"]
---

nginx从`1.9.0`开始，支持动态拓展，并且新增加了一个stream模块，用来实现四层协议的转发、代理或者负载均衡等。

<!--more-->

## 用法

stream模块用法和http模块差不多，关键的是语法几乎一致。熟悉http模块配置语法的上手更快，以下是一个配置了tcp负载均衡的例子, 有 server，upstream块，而且还有server，hash， listen， proxy_pass等指令，如果不看最外层的stream关键字，还以为是http模块呢。

    user nginx;
    worker_processes auto;
    stream {
        upstream sphinx {
        server 127.0.0.1:9312;
    }
    server {
       listen 9311;
       proxy_pass sphinx; # 把9311端口监听的数据转发到9312
     }
    }