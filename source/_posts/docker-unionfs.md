---
title: docker之联合文件系统
tags: []
id: '369'
categories:
  - - Docker
date: 2018-09-17 00:45:44
---

## 1\. 作用

联合文件系统（UnionFS） 是一种轻量级的高性能分层文件系统，它支持将文件系统中的修改信息当做一次提交，然后层层叠加（有点像git），同时可以将不同的目录挂载到同一个虚拟文件系统下，应用看到的是挂载的最终结果。

Debian/Ubuntu上成熟的AUFS（Another Union File System）就是一种联合文件系统的实现。AUFS支持为每个成员目录设定只读权限（readonly）、读写权限（readwrite）或（whiteout-able）权限，同时AUFS里有一个类似分层的概念，对只读权限的分支可以在逻辑上进行增量地修改（不影响其他只读部分）。

Docker镜像自身就是由多个文件层组成，每一层组成有唯一的编号（层ID）

<!--more-->

## 2\. docker存储

联合文件是docker镜像技术的基础。docker镜像就是根据分层技术来进行继承的。

![Alt text](/uploads/2018/09/20160819173838.png)

举个例子，用户基于一些基础镜像，来制作另外的一个镜像。这些镜像共享同一个基础镜像层，提高的存储的效率和空间利用率。

假如，我们使用php7做基础镜像，来制作多个不同的镜像，那么这些镜像，就会公用一个基础镜像作为“底层”，这样做，提高了利用率，因为不用每个自定义镜像都要创建php7的“底层”。这也就是，为什么我们再build一个镜像的时候，会把基础镜像pull下来。当我们创建的自定义镜像还要有变动的时候，至于要创建一个新的层就好了。这样，也就不用我们从头开始构建镜像，节省了构建时间。

`这也是docker十分轻量级和快速的重要原因！`

docker安装自带了查看镜像层的命令：docker history

下面我们来看下基础镜像和自定义镜像层的比较：

```bash
localhost:~ feilong$ docker pull php:7.0
7.0: Pulling from library/php
7.0: Pulling from library/php
802b00ed6f79: Pull complete
59f5a5a895f8: Pull complete
6898b2dbcfeb: Pull complete
8e0903aaa47e: Pull complete
b627a118b728: Pull complete
e2e2cb10942b: Pull complete
e63e2fa0c7d4: Pull complete
57c09353077e: Pull complete
Digest: sha256:f0e774402dd485c11c60f52c05989da088c5debb44d1126cc089970e1bfca002
Status: Downloaded newer image for php:7.0
localhost:~ feilong$
localhost:~ feilong$
localhost:~ feilong$
localhost:~ feilong$
localhost:~ feilong$
localhost:~ feilong$ docker history php:7.0
IMAGE CREATED CREATED BY SIZE COMMENT
a6c560acbfc5 9 hours ago /bin/sh -c #(nop) CMD ["php" "-a"] 0B
<missing> 9 hours ago /bin/sh -c #(nop) ENTRYPOINT ["docker-php-e… 0B
<missing> 9 hours ago /bin/sh -c #(nop) COPY multi:2cdcedabcf5a3b9… 6.42kB
<missing> 9 hours ago /bin/sh -c set -eux; savedAptMark="$(apt-m… 79.4MB
<missing> 9 hours ago /bin/sh -c #(nop) COPY file:207c686e3fed4f71… 587B
<missing> 9 hours ago /bin/sh -c set -xe; fetchDeps=' wget ';… 13.3MB
<missing> 9 hours ago /bin/sh -c #(nop) ENV PHP_SHA256=ff6f62afeb… 0B
<missing> 9 hours ago /bin/sh -c #(nop) ENV PHP_URL=https://secur… 0B
<missing> 9 hours ago /bin/sh -c #(nop) ENV PHP_VERSION=7.0.32 0B
<missing> 10 days ago /bin/sh -c #(nop) ENV GPG_KEYS=1A4E8B7277C4… 0B
<missing> 10 days ago /bin/sh -c #(nop) ENV PHP_LDFLAGS=-Wl,-O1 -… 0B
<missing> 10 days ago /bin/sh -c #(nop) ENV PHP_CPPFLAGS=-fstack-… 0B
<missing> 10 days ago /bin/sh -c #(nop) ENV PHP_CFLAGS=-fstack-pr… 0B
<missing> 10 days ago /bin/sh -c mkdir -p $PHP_INI_DIR/conf.d 0B
<missing> 10 days ago /bin/sh -c #(nop) ENV PHP_INI_DIR=/usr/loca… 0B
<missing> 10 days ago /bin/sh -c apt-get update && apt-get install… 209MB
<missing> 10 days ago /bin/sh -c #(nop) ENV PHPIZE_DEPS=autoconf … 0B
<missing> 10 days ago /bin/sh -c set -eux; { echo 'Package: php… 46B
<missing> 10 days ago /bin/sh -c #(nop) CMD ["bash"] 0B
<missing> 10 days ago /bin/sh -c #(nop) ADD file:e6ca98733431f75e9… 55.3MB
```

我pull了一个php:7.0的镜像，可以看到，整个过程分为20层，每个层级都会执行对应的命令，然后我们基于php7在做一些自定义的操作：安装mysqli和redis的扩展，构建一个新的镜像：

```bash
#Dockerfile
FROM php:7.0
RUN apt-get update \
    && docker-php-ext-install mysqli \
    && curl -L -o ./redis-4.1.0.tgz http://pecl.php.net/get/redis-4.1.0.tgz \
    && tar zxvf redis-4.1.0.tgz \
    && cd redis-4.1.0 \
    && phpize \
    && ./configure \
    && make && make install \
    && echo "extension=redis.so" > /usr/local/etc/php/conf.d/redis.ini \
    && cd .. \
    && rm -rf redis-4.1.0.tgz redis-4.1.0
```

```bash
localhost:feilong_test feilong$ docker build -t feilongtest .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM php:7.0
 ---> a6c560acbfc5
Step 2/2 : RUN apt-get update     && docker-php-ext-install mysqli     && curl -L -o ./redis-4.1.0.tgz http://pecl.php.net/get/redis-4.1.0.tgz     && tar zxvf redis-4.1.0.tgz     && cd redis-4.1.0     && phpize     && ./configure     && make && make install     && echo "extension=redis.so" > /usr/local/etc/php/conf.d/redis.ini     && cd ..     && rm -rf redis-4.1.0.tgz redis-4.1.0
 ---> Running in bd2e3fbbde25
Get:3 http://security.debian.org/debian-security stretch/updates InRelease [94.3 kB]
Get:4 http://security.debian.org/debian-security stretch/updates/main amd64 Packages [414 kB]
Ign:1 http://cdn-fastly.deb.debian.org/debian stretch InRelease
Get:2 http://cdn-fastly.deb.debian.org/debian stretch-updates InRelease [91.0 kB]
Get:5 http://cdn-fastly.deb.debian.org/debian stretch-updates/main amd64 Packages [5148 B]
Get:6 http://cdn-fastly.deb.debian.org/debian stretch Release [118 kB]
Get:7 http://cdn-fastly.deb.debian.org/debian stretch Release.gpg [2434 B]
Get:8 http://cdn-fastly.deb.debian.org/debian stretch/main amd64 Packages [7099 kB]
省略
Installing shared extensions: /usr/local/lib/php/extensions/no-debug-non-zts-20151012/
Removing intermediate container bd2e3fbbde25
 ---> 41b978fc1549
Successfully built 41b978fc1549
Successfully tagged feilongtest:latest
```

然后我们看下自己构建的镜像层

```bash
localhost:feilong_test feilong$ docker history feilongtest
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
41b978fc1549        48 seconds ago      /bin/sh -c apt-get update     && docker-php-…   18.3MB
a6c560acbfc5        10 hours ago        /bin/sh -c #(nop)  CMD ["php" "-a"]             0B
<missing>           10 hours ago        /bin/sh -c #(nop)  ENTRYPOINT ["docker-php-e…   0B
<missing>           10 hours ago        /bin/sh -c #(nop) COPY multi:2cdcedabcf5a3b9…   6.42kB
<missing>           10 hours ago        /bin/sh -c set -eux;   savedAptMark="$(apt-m…   79.4MB
<missing>           10 hours ago        /bin/sh -c #(nop) COPY file:207c686e3fed4f71…   587B
<missing>           10 hours ago        /bin/sh -c set -xe;   fetchDeps='   wget  ';…   13.3MB
<missing>           10 hours ago        /bin/sh -c #(nop)  ENV PHP_SHA256=ff6f62afeb…   0B
<missing>           10 hours ago        /bin/sh -c #(nop)  ENV PHP_URL=https://secur…   0B
<missing>           10 hours ago        /bin/sh -c #(nop)  ENV PHP_VERSION=7.0.32       0B
<missing>           10 days ago         /bin/sh -c #(nop)  ENV GPG_KEYS=1A4E8B7277C4…   0B
<missing>           10 days ago         /bin/sh -c #(nop)  ENV PHP_LDFLAGS=-Wl,-O1 -…   0B
<missing>           10 days ago         /bin/sh -c #(nop)  ENV PHP_CPPFLAGS=-fstack-…   0B
<missing>           10 days ago         /bin/sh -c #(nop)  ENV PHP_CFLAGS=-fstack-pr…   0B
<missing>           10 days ago         /bin/sh -c mkdir -p $PHP_INI_DIR/conf.d         0B
<missing>           10 days ago         /bin/sh -c #(nop)  ENV PHP_INI_DIR=/usr/loca…   0B
<missing>           10 days ago         /bin/sh -c apt-get update && apt-get install…   209MB
<missing>           10 days ago         /bin/sh -c #(nop)  ENV PHPIZE_DEPS=autoconf …   0B
<missing>           10 days ago         /bin/sh -c set -eux;  {   echo 'Package: php…   46B
<missing>           10 days ago         /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>           10 days ago         /bin/sh -c #(nop) ADD file:e6ca98733431f75e9…   55.3MB
```

可以看出，在dockerfile里面，增加了1步操作，分别是按照mysqli和redis扩展，然后在build镜像的时候，在原有的20层的基础上，继续添加了1层。

基础镜像层的层内容都是不可用自改的、只读的。当docker利用镜像创建容器的时候，会在最顶端创建一个可以读写的层给容器。容器内的数据，都会写到这个读写层里面。当所操作的对象位于比较深的层时，需要先复制到最上层的可读写层。当数据对象较大的时候，往往意味着IO性能比较差。因此，一般推荐奖容器修改数据通过volume方式挂载，而不是直接修改镜像内的数据。

Docker的所有存储，都是在docker文件夹下面，以Centos或者Ubuntu为例，默认的路径一般是/var/lib/docker。（我仅仅以Centos为例）

```bash
[root@izj6c9b96ia369l2i47yq3z docker]# ll
total 52
drwx------ 2 root root 4096 Sep  6 20:09 builder
drwx------ 4 root root 4096 Sep  6 20:09 buildkit
drwx------ 3 root root 4096 Sep  6 20:09 containerd
drwx------ 2 root root 4096 Sep 17 00:07 containers
drwx------ 3 root root 4096 Sep  6 20:09 image
drwxr-x--- 3 root root 4096 Sep  6 20:09 network
drwx------ 4 root root 4096 Sep 17 00:07 overlay2
drwx------ 4 root root 4096 Sep  6 20:09 plugins
drwx------ 2 root root 4096 Sep  6 20:09 runtimes
drwx------ 2 root root 4096 Sep  6 20:09 swarm
drwx------ 2 root root 4096 Sep 16 23:56 tmp
drwx------ 2 root root 4096 Sep  6 20:09 trust
drwx------ 2 root root 4096 Sep  6 20:55 volumes
```

docker的镜像层基本上都是在overlay2里面

```bash
[root@izj6c9b96ia369l2i47yq3z overlay2]# docker images
REPOSITORY TAG IMAGE ID CREATED SIZE
busybox latest e1ddd7948a1c 6 weeks ago 1.16MB
[root@izj6c9b96ia369l2i47yq3z docker]# cd overlay2/
[root@izj6c9b96ia369l2i47yq3z overlay2]# ll
total 8
drwx------ 3 root root 4096 Sep 16 23:56 4c819c3673c3416b65c2cf6394818d270363cfd53a0389a5f6c237e1c8ad3ef4
drwxr-xr-x 2 root root 4096 Sep 17 00:07 l
```

现在，我们只有一个busybox的镜像，该目录下面包括diff文件夹，diff文件夹就是我们创建容器之后，初始化的文件夹

```bash
[root@izj6c9b96ia369l2i47yq3z diff]# ll
total 40
drwxr-xr-x 2 root      root      12288 Aug  1 04:20 bin
drwxr-xr-x 2 root      root       4096 Aug  1 04:20 dev
drwxr-xr-x 3 root      root       4096 Aug  1 04:20 etc
drwxr-xr-x 2 nfsnobody nfsnobody  4096 Aug  1 04:20 home
drwx------ 2 root      root       4096 Aug  1 04:20 root
drwxrwxrwt 2 root      root       4096 Sep 17 00:06 tmp
drwxr-xr-x 3 root      root       4096 Aug  1 04:20 usr
drwxr-xr-x 4 root      root       4096 Aug  1 04:20 var
[root@izj6c9b96ia369l2i47yq3z diff]#
```

为了验证我们说的是否是正确的，我们在tmp的文件夹里面创建一个测试的文件a.txt，然后写入Hello world

```bash
[root@izj6c9b96ia369l2i47yq3z diff]# touch  tmp/a.txt
[root@izj6c9b96ia369l2i47yq3z diff]# echo 'Hello world' > tmp/a.txt
[root@izj6c9b96ia369l2i47yq3z diff]# cat tmp/a.txt
Hello world
[root@izj6c9b96ia369l2i47yq3z diff]#
```

如果分析是正确的，那么创建的容器中，也会存在这个文件

```bash
[root@izj6c9b96ia369l2i47yq3z diff]# docker run -it --rm --name test busybox
/ # ll
sh: ll: not found
/ # ls -l
total 36
drwxr-xr-x    2 root     root         12288 Jul 31 20:20 bin
drwxr-xr-x    5 root     root           360 Sep 16 16:19 dev
drwxr-xr-x    1 root     root          4096 Sep 16 16:19 etc
drwxr-xr-x    2 nobody   nogroup       4096 Jul 31 20:20 home
dr-xr-xr-x  131 root     root             0 Sep 16 16:19 proc
drwx------    1 root     root          4096 Sep 16 16:19 root
dr-xr-xr-x   13 root     root             0 Sep 16 16:19 sys
drwxrwxrwt    2 root     root          4096 Sep 16 16:17 tmp
drwxr-xr-x    3 root     root          4096 Jul 31 20:20 usr
drwxr-xr-x    4 root     root          4096 Jul 31 20:20 var
/ # ls -l tmp
total 4
-rw-r--r--    1 root     root            12 Sep 16 16:17 a.txt
/ # cat tmp/a.txt
Hello world
/ #
```

在创建的容器中，我们果然看到了内容为Hello world的tmp/a.txt文件

在创建容器之后，我们会发现多了两个文件夹

![Alt text](/uploads/2018/09/WX20180917-002724@2x-1024x494.png)

这两个文件夹，是用来存储一些容器的数据，如果容器一旦删除，那么这些数据也会随着一块被清理掉，这就是为什么建议我们把一些重要的数据，挂载到外部的原因！

![Alt text](/uploads/2018/09/WX20180917-003010@2x-1024x368.png)

## 3\. 多种文件系统比较

Docker目前支持多种联合文件系统：AUFS、OverlayFS、btrfs、vfs、zfs和Device Mapper等。

AUFS：最早支持的文件系统，对Debian/Ubuntu支持比较好，虽然没有合并到Linux内核，但是成熟度很高

OverlayFS：类似AUFS，性能更好，上面的例子明显就是OverlayFS，已经合并到内核，将来会取代AUFS

Device Mapper：Redhat和Docker团队一起开发并用于支持RHEL的文件系统，内核支持，性能略慢，成熟度高

...

## 4\. 总结

docker的镜像层级设计，让docker的性能更高，更加符合软件设计，具有很高的复用性，这个也是docker镜像编译迅速的重要原因。

此外，docker容器默认将数据存储到docker文件夹下，如果容器被删除，那么容器数据也将被删除掉，所以，对于容器的重要数据，我们应该映射到宿主机上面，避免由于容器删除，而导致的数据丢失。

另：关于镜像层ID为missing，请参阅论坛：[Layer IDs shown as \\<missing> in history](https://forums.docker.com/t/layer-ids-shown-as-missing-in-history/6325)