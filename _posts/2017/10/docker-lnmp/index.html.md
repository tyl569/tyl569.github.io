---
layout: post
title: 使用docker搭建lnmp环境
date: 2017-10-29
tags: ["docker","Docker","Linux","PHP","容器"]
---

docker是一个开源的容器引擎，随着"微服务架构"正在变得越来越重要，docker也变得越来越火。但是网上的文章中，要么是很有借鉴意义的干货，要么就是使用高端术语来讲述什么叫做微服务架构。今天我就通过文章来记述一下传统lnmp迁移docker的过程。

<!--more-->
<!-- TOC -->

*   [项目背景](#项目背景)
*   [前期准备](#前期准备)

    *   [安装docker](#安装docker)
    *   [获取php-fpm镜像](#获取php-fpm镜像)
    *   [同理获取nginx镜像和MySQL镜像](#同理获取nginx镜像和mysql镜像)
    *   [端口检查](#端口检查)
    *   [查看镜像](#查看镜像)
    *   [运行镜像](#运行镜像)
*   [镜像配置](#镜像配置)
*   [总结](#总结)
<!-- /TOC -->

#### 项目背景

主要是以自身的博客系统作为迁移的样例，项目环境是传统的lnmp环境。

#### 前期准备

##### 安装docker

    $ sudo apt-get install docker.io -y
    Reading package lists... Done
    Building dependency tree
    Reading state information... Done
    ...
    Setting up docker.io (1.6.2~dfsg1-1ubuntu4~14.04.1) ...
    docker start/running, process 26908

##### 获取php-fpm镜像

    $ sudo docker search php-fpm #查找php-fpm镜像
    NAME                              DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
    php                               While designed for web development, the PH...   2782      [OK]
    richarvey/nginx-php-fpm           Container running Nginx + PHP-FPM capable ...   454                  [OK]
    bitnami/php-fpm                   Bitnami PHP-FPM Docker Image                    41                   [OK]
    phpdockerio/php7-fpm
    ...

    $ sudo docker pull  phpdockerio/php7-fpm:latest
    latest: Pulling from phpdockerio/php7-fpm
    632d62e9ff45: Pull complete
    4719c3e8a982: Pull complete
    2309d29c605a: Pull complete
    83aeee240cf5: Pull complete
    6962aaa46258: Pull complete
    ceb4c4ec812a: Pull complete
    821e3516e882: Pull complete
    ef64564fd4f8: Pull complete
    4ce8803d2ea8: Pull complete
    ba9d4bc26f3e: Pull complete
    20fd756c6431: Pull complete
    f7729a02ff06: Pull complete
    Digest: sha256:a2a240a31c8afdf723a8554b6c46691069a80ac622cbb5ab77fcd7b5762ddc58
    Status: Downloaded newer image for phpdockerio/php7-fpm:latest

##### 同理获取nginx镜像和MySQL镜像

    $ sudo docker pull nginx:latest
    $ sudo docker pull mysql:latest

##### 端口检查

    $ netstat -anp ' grep "80\'3306\'9000" ##查看80 3306 和 9000端口占用情况，如果被占用，停掉响应服务

##### 查看镜像

    $ sudo docker images
    REPOSITORY             TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    mysql                  latest              3ad8e8e4bdb1        14 hours ago        408.2 MB
    phpdockerio/php7-fpm   latest              f7729a02ff06        5 days ago          166.2 MB
    nginx                  latest              2ecc072be0ec        7 days ago          108.3 MB

##### 运行镜像

生成MySQL容器

    $ sudo docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=密码 --name docker_mysql_00 mysql:latest

生成nginx容器，外部80端口映射到内部80端口，关联容器内外文件夹

    $ sudo docker run -d -p 80:80 -v /usr/share/nginx:/usr/share/nginx -v /etc/nginx:/etc/nginx --name docker_nginx_00 nginx:lastest

生成php-fpm容器, 同理

    $ sudo docker run -d -p 9000:9000 -v /usr/share/nginx:/usr/share/nginx --name docker_php_fpm_00 phpdockerio/php7-fpm

#### 镜像配置

查看容器的ip

    $ sudo docker inspect docker_php_fpm_00 docker_nginx_00 docker_mysql_00' grep "IPAddress"
    "IPAddress": "172.17.0.7",
    "IPAddress": "172.17.0.3",
    "IPAddress": "172.17.0.2",

配置nginx和php-fpm

    $ sudo docker exec -ti docker_nginx_00 /bin/bash #进入docker_nginx_00容器
    vim /etc/nginx/sites-enabled/blog.conf ##像正常一样配置nginx，

    server {
            listen 80;
            root /usr/share/nginx/wordpress;
            index index.php index.html index.htm index.nginx-debian.html;

            server_name feilong.tech www.feilong.tech feilong.tech;

            location ~ \.php$ {
                    fastcgi_pass 172.17.0.7:9000; ##php-fpm容器的IP
                    fastcgi_index  index.php;
                    set $path_info "";
                    set $real_script_name $fastcgi_script_name;
                    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
                    include        fastcgi_params;

            }
    }

同理配置好MySQL的IP地址，容器可能没有安装vim，所以编辑之前需要提前`apt-get update`。然后进行安装。

#### 总结

安装过程比较复杂，尤其是需要配置IP。其实整个过程并不是符合docker的期望，理想情况是将lnmp放到一个容器中，即直接使用`sudo docker search lnmp`查找镜像，进行意见安装。前端使用nginx，通过\$host配置转发到端口，然后通过docker端口的映射到达容器内部。