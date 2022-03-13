---
title: docker之libnetwork插件化网络功能
tags:
  - docker
  - docker network
  - libnetwork
id: '402'
categories:
  - - Docker
date: 2018-10-02 23:06:07
---

# 1\. 背景

docker的版本不断迭代，从1.7开始，docker将网络的部分以插件化的方式剥离出来，允许用户通过命令实现不同的后端，增加了docker的灵活性。剥离出来的网络叫做libnetwork，显而易见，它希望将来为不同的容器定义统一网络层的规范。

<!--more-->

# 2\. 容器的网络模型（CNM）

容器网络模型（Container Networking Model，CNM）最开始的提出者是思科公司员工Erik，设计了一个docker扩展的网桥模型。后来，他将这个模型反馈给docker的团队，得到了docker的极大的认可，也是libnetwork的雏形。

CNM主要包含三点sandbox、endpoint和network。

之前有篇文章是[DOCKER之命名空间](http://feilong.tech/2018/09/10/docker-namespace/) ，文章简要说了一下网络的隔离，讲述了一些docker的网络基本知识。其实也是libnetwork的一个缩影。

sandbox相当于网卡配置、路由表、dns配置、网络命名空间等

endpoint相当于容器内部的虚拟网卡（eth0等）

network代表一个或者多个网络

当然，这些概念并不是对等，这里只是有一个类比的关系。


![](/uploads/2018/09/mp55072270_1453099798138_1_th.jpeg)

可见，具体底层网络如何实现，每个容器的endpoint所连接的network，以及sandbox如何进行隔离，都可以不用关心。只要libnetwork提供网络和接入点，只需要容器给街上或者拔下（这里指的是endpoint和network的链接和断开），剩下的都是直接由驱动自己实现。

# 3\. docker网络bridge

docker自带了查看网络的命令

```bash
localhost:~ feilong$ docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
5adfcce2b464        bridge                bridge              local
c2d85093b647        docker_default        bridge              local
0eec2b9c7d57        go_default            bridge              local
604e2a13b930        host                  host                local
bded596e1021        marvin2_default       bridge              local
6c006e90b785        marvin_code-network   bridge              local
ef31b0632a2f        marvin_default        bridge              local
72169048d0e7        none                  null                local
6eb2e51b63e2        redis_default         bridge              local
6c6e4cc36312        registry_default      bridge              local
localhost:~ feilong$
```

从查询结果，可以看出，我的电脑上面，有很多的网络信息。docker默认是携带3种网络：bridge、host、none，这三个网络是不能被删除的

```bash
localhost:~ feilong$ docker network rm host
Error response from daemon: host is a pre-defined network and cannot be removed
localhost:~ feilong$ docker network rm bridge
Error response from daemon: bridge is a pre-defined network and cannot be removed
localhost:~ feilong$ docker network rm none
Error response from daemon: none is a pre-defined network and cannot be removed
localhost:~ feilong$
```

我们在运行容器的时候，指定的网络类型（--network=bridge--network=host--network=none）其实就是来源于这里，又由于docker默认是使用bridge的网络类型，通常是省略这个参数。

#### 使用docker-compose发生了什么？

我们在使用docker-compose的时候，通常是不用设置--link参数的，那是因为，docker会认为使用docker-compose启动多个容器，但是这几个容器是组成了一个service，会自动创建一个bridge类型的网络，然后将启动的容器，放到这个网络中，和其他的网络进行隔离。

```yaml
# docker-compose.yaml
version: '3'

services: 
  busybox1:
    image: busybox
    container_name: busybox1
  busybox2:
    image: busybox
    container_name: busybox2
  busybox3:
    image: busybox
    container_name: busybox3
```

查看docker的网络

```bash
localhost:test feilong$ docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
5adfcce2b464        bridge                bridge              local
c2d85093b647        docker_default        bridge              local
0eec2b9c7d57        go_default            bridge              local
604e2a13b930        host                  host                local
bded596e1021        marvin2_default       bridge              local
6c006e90b785        marvin_code-network   bridge              local
ef31b0632a2f        marvin_default        bridge              local
72169048d0e7        none                  null                local
6eb2e51b63e2        redis_default         bridge              local
6c6e4cc36312        registry_default      bridge              local
localhost:test feilong$
```

运行docker-compose，然后查看网络

```bash
localhost:test feilong$ docker-compose up -d
Creating network "test_default" with the default driver
Creating busybox2 ... done
Creating busybox3 ... done
Creating busybox1 ... done
localhost:test feilong$ docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
5adfcce2b464        bridge                bridge              local
c2d85093b647        docker_default        bridge              local
0eec2b9c7d57        go_default            bridge              local
604e2a13b930        host                  host                local
bded596e1021        marvin2_default       bridge              local
6c006e90b785        marvin_code-network   bridge              local
ef31b0632a2f        marvin_default        bridge              local
72169048d0e7        none                  null                local
6eb2e51b63e2        redis_default         bridge              local
6c6e4cc36312        registry_default      bridge              local
fca18d6c1259        test_default          bridge              local
```

从上面，我们看出，在运行docker-compose的时候，就已经创建了一个网络，叫做 test\_default

#### \--link ？

我们知道，想要将两个容器链接起来，可以使用--link参数（在使用命令行 docker run的时候）。其实，我们在进行docker run的时候，创建的容器，加入了一个默认的网络“bridge”，但是在“bridge”中，默认是不能进行连接的，需要加上--link，使bridge内部的容器进行连接。这样也是为了隔离容器。试想一下，如果不采用这个机制，那么网络空间就不再是隔离的状态了，就和docker的设计不太相符。

我们分别运行一个test1和test2容器

```bash
feilongdeMacBook-Pro:~ feilong$ docker run -itd --name test1 busybox
bfca24444ce4ee16d6e2af76cf0967d0923dbbed4310c811cdbefef5d6b5b67f
feilongdeMacBook-Pro:~ feilong$ docker run -itd --name test2 busybox
48a5c7e9c042de472ed5edd328a53952fe9552357eff7b1c81f4d9e3bbed1c8c
feilongdeMacBook-Pro:~ feilong$ docker ps  grep test
48a5c7e9c042        busybox                              "sh"                     6 seconds ago       Up 5 seconds                                  test2
bfca24444ce4        busybox                              "sh"                     10 seconds ago      Up 9 seconds                                  test1
b742b60be6a0        registry:latest                      "/entrypoint.sh /etc…"   22 hours ago        Up 3 hours          0.0.0.0:32768->5000/tcp   thirsty_almeida
feilongdeMacBook-Pro:~ feilong$
```

然后我们进入test1，ping一下test2

```bash
feilongdeMacBook-Pro:~ feilong$ docker exec -it test1 sh
/ # ping test2
ping: bad address 'test2'
/ #
```

结果很明显，是不通的！

但是他们是在同一个网络里面的

![](/uploads/2018/10/1__bash.jpg) ![](/uploads/2018/10/2__bash.jpg)

# 4\. 总结

libnetwork通过CNM，抽象了下层网络的实现让docker可以根据需要，实现不同的网络技术，docke将swarm引擎加到了新的版本中，以提供集群网络的更好支持。