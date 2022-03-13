---
title: Ubuntu搭建lnmp环境
tags: []
id: '143'
categories:
  - - Linux
date: 2017-08-24 20:20:18
---

**项目基于php CI**

#### 1、安装mysql

```bash
$ sudo apt-get install mysql-server mysql-client
```

<!-- more -->

**安装过程会提示输入root的密码，连续输入两次**

```bash
New password for the MySQL “root” user: <– 输入你的密码
Repeat password for the MySQL “root” user: <– 再输入一次
```

#### 2、安装nginx

**安装之前看看是否安装了Apache，如果安装了，先停掉Apache，防止端口占用**

```bash
$ sudo service apache2 stop
$ sudo update-rc.d -f apache2 remove
$ sudo apt-get remove apache2
$ sudo apt-get install nginx
$ sudo service nginx start 
```

**试试安装是否成功，在浏览器输入IP或主机地址。**

#### 3、安装php

```bash
$ sudo apt-get install php7.0 -y
```

**启动php-fpm**

```bash
$ sudo service php7.0-fpm start
```

**输入`php -i`命令，查看php是否运行, 这个命令和`phpinfo()`函数一样**

```bash
$ sudo php -i
```

#### 4、更改nginx配置文件

```bash
$ cd /etc/nginx/sites-enabled/
$ ll
total 8
drwxr-xr-x 2 root root 4096 Nov 22 08:20 ./
drwxr-xr-x 7 root root 4096 Nov 22 08:21 ../
lrwxrwxrwx 1 root root   34 Nov 22 06:10 default -> /etc/nginx/sites-available/default
```

**这里会有一个默认的default配置文件，更改配置文件，进行项目配置**

```bash
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_tokens off;
        root /usr/share/nginx/test;
        # Add index.php to the list if you are using PHP
        index  index.php index.html index.htm index.nginx-debian.html;
        server_name _;
        if ($request_filename !~ (\.jpgcssjspngfontsimg)) {
                rewrite ^/(.*)$ /index.php/$1 break;
        }
        location ~ / {
                root /usr/share/nginx/test;
                fastcgi_pass   unix:/var/run/php/php7.0-fpm.sock;
                fastcgi_index  index.php;
                set $path_info "";
                set $real_script_name $fastcgi_script_name;
                fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
                include        fastcgi_params;
        }
}
```

**按需配置CI框架的数据库**

**新建控制器`Info.php`, 在里面增加php方法，运行检验**

```php
<?php if ( ! defined('BASEPATH')) exit('No direct script access allowed');

class Info extends CI_Controller {
    public function php() {
        echo "hello world!";
    }
}
```

**发现抛出一个致命的错误**

`Fatal error: Uncaught TypeError: Argument 1 passed to CI_Exceptions::show_exception() must be an instance of Exception, instance of Error given, called in /usr/share/nginx/test/system/core/Common.php on line 659 and defined in /usr/share/nginx/test/system/core/Exceptions.php:192 Stack trace: #0 /usr/share/nginx/test/system/core/Common.php(659): CI_Exceptions->show_exception(Object(Error)) #1 [internal function]: _exception_handler(Object(Error)) #2 {main} thrown in /usr/share/nginx/test/system/core/Exceptions.php on line 192`

**报错的意思大概是说show\_exception方法的参数是个实例化的，但是传入的参数不是一个实例**

**后来根据github上面的解决办法https://github.com/bcit-ci/CodeIgniter/commit/84f24c23baf5ea45c30c4ab3cbc57cd846ea0f56#diff-e57013406b22fdf1ec021ddda0d8e5aa，更改show\_exception的参数**

**刷新后发现提示mysql的php拓展，使用命令安装php相关拓展**

#### 5、安装php拓展

**查看支持的拓展**

```bash
sudo apt-cache search php7.0

libapache2-mod-php7.0 - server-side, HTML-embedded scripting language (Apache 2 module)
php-all-dev - package depending on all supported PHP development packages
php7.0 - server-side, HTML-embedded scripting language (metapackage)
php7.0-cgi - server-side, HTML-embedded scripting language (CGI binary)
php7.0-cli - command-line interpreter for the PHP scripting language
php7.0-common - documentation, examples and common module for PHP
php7.0-curl - CURL module for PHP
php7.0-dev - Files for PHP7.0 module development
php7.0-gd - GD module for PHP
php7.0-gmp - GMP module for PHP
php7.0-json - JSON module for PHP
php7.0-ldap - LDAP module for PHP
php7.0-mysql - MySQL module for PHP
php7.0-odbc - ODBC module for PHP
php7.0-opcache - Zend OpCache module for PHP
php7.0-pgsql - PostgreSQL module for PHP
php7.0-pspell - pspell module for PHP
.
.
.
```

**为了保险起见，直接安装所有拓展**

```bash
sudo apt-get install php-all-dev
```

**然后刷新发现正常输出`hello world!`**

**然后我把show\_exception方法又改回原来的样子，刷新之后没有再出现该情况，则证明show\_exception也是由于缺少拓展造成的**