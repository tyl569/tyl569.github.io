---
title: 使用Nginx实现流量限流
tags:
  - http
id: '214'
categories:
  - - Nginx
comments: false
date: 2018-01-25 23:47:21
---

Nginx是高性能的http服务器，同时也可以作为一个反向代理的服务器，甚至还可以作为一个IMAP/pop3/SMTP服务器。Nginx除了负责请求的负载均衡和分发等工作外，自带的限流模块也可以帮助运维人员限制流量的速率。

<!--more-->

#### 更改配置

开启Nginx的限流功能

```nginx
http {
    #定义每个IP的session空间大小
    limit_conn_zone $binary_remote_addr zone=one:20m;
    #与limit_conn_zone类似，定义每个允许发起的请求速率
    limit_req_zone $binary_remote_addr zone=req_one:20m rate=1r/s;
    #定义每个IP发起的并发连接数
    limit_conn one 10;
    #缓存还没来得及处理的请求
    limit_req zone=req_one burst=100;

    #rewrite_log on;
    #error_log /var/log/nginxrewrite.log notice;
        client_body_buffer_size 256M;
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" "$request_filename"';
        sendfile        on;

        keepalive_timeout  65;
        include servers/*.conf;
}
```

#### 进行验证

我们使用Linux或者mac自带的`ab`命令进行验证，并且实时查看access.log。在Nginx配置文件中，我们设置的请求速率是每秒1个请求，那我们则需要设置每秒超多1个请求才行

```bash
$ ab -n 20 http://sdeno-api/info/php
```

![](/uploads/2018/01/nginx_limit_request00.png)

从截图的日志中，我们可以看出Nginx限流模块确实可以限制请求速率。

#### 其他

由于nginx的版本问题，旧版本会limit\_conn\_zone->limit\_zone