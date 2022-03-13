---
title: docker之控制组
tags:
  - docker
id: '361'
categories:
  - - Docker
date: 2018-09-12 18:55:50
---

### 1\. 作用

控制度(CGroups) 其实是Linux内核的一个特性，主要是用来控制共享资源，比如限制内存、CPU的的一些使用等。容器使用的CPU、内存等硬件信息，其实就是使用的宿主机上面的硬件设备，所以合理的分配资源，也是为了避免不同容器之间、容器和宿主机进程之间，产生资源的抢占。

## 2\. 容器控制组

### 2.1 资源限制

比如我们要限制容器的使用内存，可以在run的时候加上--memory的参数

```bash
feilongdeMBP:~ feilong$ docker run -it --rm --name test --memory 10m busybox
```

然后新打开一个窗口，可以实时查看下容器的内存使用情况

![](/uploads/2018/09/WX20180912-000121.png)

TIP:

使用docker-compose的时候需要注意一下，设置内存限制的参数是mem\_limit，但是在docker-compose的3.x版本之后，不支持这个参数，所以在写docker-compose.yaml 的时候，会出现 Unsupported config option for xxxx: 'mem\_limit' 的错误信息，所以需要指定 version: '2'

### 2.2 优先级

docker run的时候支持使用-c 或者 --cpu-shares 用来指定容器使用CPU的加权值。如果不指定，那么就是使用的是默认值，一般是1024。

\-c 或者 --cpu-shares并不能指定容器能够使用多少CPU或者多少GHz，而是一个加权值。有点类似nginx的负载均衡配置。

这个配置在少量容器的时候，并没有太大的实际意义。只有CPU资源比较紧缺的时候，这个配置参数才会展现出来。

比如，一个容器的加权值是100，另一个加权值是50，那么加权值为100的容器，获取CPU时间片的概率就是另一个的2倍

如果只有一个容器，那么CPU时间片肯定都会给这个容器使用。

创建一个容器，安装stress软件，然后开启10个进程，看下CPU占用情况

```bash
localhost:marvin feilong$ docker run -itd --name cpu512  --cpu-shares 512 ubuntu
localhost:marvin feilong$ docker exec -it cpu512
# apt update
# apt install stress
# stress -c 10
stress: info: [250] dispatching hogs: 10 cpu, 0 io, 0 vm, 0 hdd
```

打开一个新窗口，然后登录到这个容器，使用top查看下CPU占用情况

```bash
localhost:~ feilong$ docker exec -it cpu512 sh
```

![](/uploads/2018/09/WX20180912-185154.png)

可以从截图看到cpu大概占用了3.3%左右

新打开另一个窗口，创建新容器，一样的操作，安装stress，然后开10个进程，查看下CPU占用情况

```bash
localhost:~ feilong$ docker run -itd --name cpu1024  --cpu-shares 1024 ubuntu
# apt update
# apt install stress
# stress -c 10
stress: info: [241] dispatching hogs: 10 cpu, 0 io, 0 vm, 0 hdd
```

新开窗口，进入cpu1024容器，然后使用top查看CPU占用情况

![Alt text](/uploads/2018/09/WX20180912-185440.png)

可以看出CPU占用情况大概是6.6%左右，基本上是cpu是cpu512的两倍。

### 2.3 资源审计

资源审计主要是做一些审计操作，用来统计系统实际上把多少资源用到适合的目的上，可以使用cpuacct子系统记录某个进程组使用的CPU时间

### 2.4 隔离

为组隔离命名空间，这样一个组不会看到另一个组的进程、网络连接和文件系统

### 2.5 控制

挂起、恢复和重启动等操作