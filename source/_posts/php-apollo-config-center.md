---
title: PHP配合使用携程Apollo配置中心
tags:
  - Apollo
  - devops
  - 微服务
id: '411'
categories:
  - - Docker
  - - Linux
  - - PHP
date: 2018-10-27 16:41:39
---

# 1\. 背景

平时开发最头疼之一就是各种配置：

1.  一个项目往往会包含各式各样的配置信息，且不说数据库、redis、memcache这些常用的配置，还会有很多业务上的配置。
2.  线上、测试和开发环境配置各不一样，每个环境都要保存一份
3.  每次上线的时候，都要挨个check一下，
4.  更改某个配置，需要重新上线代码
5.  ￼....

所以配置中心，在devops的开发中，是必不可少的，配置中心，也可以有效的避免，因为更改配置代码，导致的代码运行出错的风险。

# 2\. Apollo

携程的[Apollo](https://github.com/ctripcorp/apollo)配置中心，在业内算是比较有名的，github上面大概有8.5k的star。很多知名的公司也都在使用。至于实现的原理直接看github上面的wiki即可。

# 3\. DO IT

## 3.1 创建项目

3.1.1 创建项目，clone项目

![](/uploads/2018/10/WX20181027-143049@2x.png)

3.1.2 按需更改docker-compose.yml

```bash
feilong$ cd scripts/docker-quick-start/
feilongdeMacBook-Pro:docker-quick-start feilong$ ll
total 8
-rw-r--r--  1 feilong  wheel  663 Oct 27 14:56 docker-compose.yml
drwxr-xr-x  4 feilong  wheel  128 Oct 27 14:56 sql
```

因为我不是java技术栈，所以以docker运行。

本地由于8080端口被占用，所以把端口改为8082

```yaml
# docker-compose.yml
version: '2'

services:
  apollo-quick-start:
    image: nobodyiam/apollo-quick-start
    container_name: apollo-quick-start
    depends_on:
      - apollo-db
    ports:
      - "8082:8080"
      - "8070:8070"
    links:
      - apollo-db

  apollo-db:
    image: mysql:5.7
    container_name: apollo-db
    environment:
      TZ: Asia/Shanghai
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
    depends_on:
      - apollo-dbdata
    ports:
      - "13306:3306"
    volumes:
      - ./sql:/docker-entrypoint-initdb.d
    volumes_from:
      - apollo-dbdata

  apollo-dbdata:
    image: alpine:latest
    container_name: apollo-dbdata
    volumes:
      - /var/lib/mysql
```

通过docker-compose启动Apollo服务

```bash
feilongdeMacBook-Pro:docker-quick-start feilong$ docker-compose up -d
Creating network "docker-quick-start_default" with the default driver
Creating apollo-dbdata ... done
Creating apollo-db     ... done
Creating apollo-quick-start ... done
feilongdeMacBook-Pro:docker-quick-start feilong$
```

访问地址 localhost:8070，如果启动失败的话，请参考 [#1473 docker\_quick\_start 起不来](https://github.com/ctripcorp/apollo/issues/1473)

![](/uploads/2018/10/WX20181027-152632.png)

## 3.2 配置

Apollo内置了账号 _**apollo/admin**_，登录之后，可以看到有个默认的应用SampleApp

![](/uploads/2018/10/WX20181027-153023.png)

通过后台创建一个用于演示的redis的配置信息

![](/uploads/2018/10/WX20181027-153143@2x.png) ![](/uploads/2018/10/1540625597036.jpg)

创建结束后，点击“发布”按钮，发布最新的配置

## 3.3 运行测试

除了配置的server端，还要有接受配置的client端，我是PHP技术栈，所以就以PHP为主

### 3.3.1 初始化项目

配置php项目，通过_composer_引入Apollo的SDK

```bash
feilongdeMacBook-Pro:apollo feilong$ composer require multilinguals/apollo-client --ignore-platform-reqs
Using version ^0.1.1 for multilinguals/apollo-client
./composer.json has been created
Loading composer repositories with package information
Updating dependencies (including require-dev)
Package operations: 1 install, 0 updates, 0 removals
 - Installing multilinguals/apollo-client (v0.1.1): Downloading (100%)
Writing lock file
Generating autoload files
```

![](/uploads/2018/10/WX20181027-161420@2x.png)

新建一个_pull.php_，为了能够实时获取最新的配置，需要长时间运行。

```php
<?php
require_once 'vendor/autoload.php';

use Org\Multilinguals\Apollo\Client\ApolloClient;

// docker-compose里面配置的API服务的端口
$serverIp = '192.168.1.72:8082';
// 在Apollo的后台可以查到
$appId = 'SampleApp';
$namespaces = array('application');
$apollo = new ApolloClient($serverIp, $appId, $namespaces);
$apollo->save_dir = 'config';

$restart = true; //auto start if failed
do {
    $error = $apollo->start();
    if ($error) {
        echo('error:'.$error."\n");
    }
} while ($error && $restart);
```

### 3.3.2 运行项目

新建一个config的目录，运行pull.php文件，获取配置信息

![](/uploads/2018/10/WX20181027-162049@2x.png)

新建一个窗口，查看下config文件夹下的文件

![](/uploads/2018/10/WX20181027-162156@2x.png)

发现新建了一个配置文件，里面是相应的配置信息

后台做一些更新操作

![](/uploads/2018/10/WX20181027-162346.png) ![](/uploads/2018/10/WX20181027-162403.png)

发布配置，查看配置文件

![](/uploads/2018/10/WX20181027-162428@2x.png)

# 4\. 总结

通过配置中心，我们就不用在部署的时候，手动一直更改项目的配置文件，可以实现自动化，降低人为风险。

此外，Apollo是版本控制的，支持回滚操作，这样，就算是出现手误，也能及时回滚配置，及时生效。