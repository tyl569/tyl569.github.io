---
title: 关于limit_req和limit_conn的区别
tags:
  - http
  - nginx
  - tcp
id: '217'
categories:
  - - Linux
  - - Nginx
comments: false
date: 2018-01-26 23:34:13
---

#### request和connection区别

在nginx里面，limit\_req和limit\_conn都是用来限流的但是两者不在一个层次上，在此之前，需要先清楚request和connect的区别。 request是请求，是http层面的。connection是链接，指的是tcp层面。所以，从含义上面可以知道两者不在一个层次。 我们在打开一个网页的时候，请求过程一般就是先经过tcp三次握手，然后在进行http请求。也就是一个connection建立之后，可以有很多request。

一个connection建立，只要服务器处理的过来，可以处理任意多的request都是没有问题的

好了，现在知道区别了。

<!--more-->

#### limit\_conn

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=one:20m;
    limit_conn one 1; #最多可以进行1个connection
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

上面的配置文件的含义很明白，nginx只接受最多一个connect，我们使用ab命令查看下

```bash
$ ab -n10  http://sdeno-api/info/php #默认通过一个connect送10个request
```

![](/uploads/2018/01/limit_conn_00.png)

从截图看来，在一个并发下，处理10个request下完全没有问题

接下来我们使用2个并发，2个请求试下，也就是两个用户，每个用户发送一个request

```bash
$ ab -n2 -c2 http://sdeno-api/info/php
```

![](/uploads/2018/01/limit_conn_01.png)

从截图中可以看到，由于nginx设置了至多限制一个并发，所以导致两个用户的请求只能有一个被处理掉，另外一个返回http 503

#### limit\_req

下面更改下nginx.conf配置文件

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=one:20m;
    limit_req_zone $binary_remote_addr zone=req_one:20m rate=1r/s;
    limit_conn one 20;
    limit_req zone=req_one burst=5;
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

上面的配置的含义是请求速率限制为1r/s，然后再缓存5个request，然后再依次处理请求（令牌桶算法）

```bash
$ ab -n10 -c10 http://sdeno-api/info/php
```

![](/uploads/2018/01/limit_conn_02-1.png)

我们使用压测命令，10个并发，10个请求，根据配置的文件，当有请求过来，nginx先处理一个请求，然后将5个请求缓存下来(burst=5)，根据设置的速率1r/s进行处理，也就是一共能够处理6个请求，其余的请求则被丢掉。

接下来我们再继续更改下nginx配置文件

```bash
$ ab -n10 -c10 http://sdeno-api/info/php
```

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=one:20m;
    limit_req_zone $binary_remote_addr zone=req_one:20m rate=1r/s;
    limit_conn one 20;
    limit_req zone=req_one burst=5 nodelay;
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

![](/uploads/2018/01/limit_conn_03.png)

咦？增加了nodelay好像和不加有点不同。这是因为请求过来的时候，会爆发出一个峰值的处理能力，处理的总的请求数是burst+1，其余的请求丢弃。

#### 总结

*   request和connect是完全不同层面的含义，一个属于http，一个属于tcp
    
*   `limit_req zone=req_zone burst=5` 依照在limti\_req\_zone中配置的rate来处理请求，同时设置了一个大小为5的缓冲队列，在缓冲队列中的请求会等待慢慢处理，超过了burst缓冲队列长度和rate处理能力的请求被直接丢弃，表现为对收到的请求有延时
    
*   `limit_req zone=req_zone burst=5 nodelay` 依照在limti\_req\_zone中配置的rate来处理请求，同时设置了一个大小为5的缓冲队列，当请求到来时，会爆发出一个峰值处理能力，对于峰值处理数量之外的请求，直接丢弃。
    

#### 参考文献

*   [Nginx下limit\_req模块burst参数超详细解析](http://blog.csdn.net/hellow__world/article/details/78658041)