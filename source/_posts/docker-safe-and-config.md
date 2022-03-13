---
title: Docker之安全防护与配置
tags:
  - docker
  - docker安全
id: '396'
categories:
  - - Docker
date: 2018-09-24 00:49:34
---

## 1\. 风险的来源

docker的安全性，在很大程度上来自于Linux的本身。目前，我们考虑到的安全性时，主要考虑下面几个方面：

*   Linux内核的命名空间机制提供的容器隔离安全
*   Linux控制组机制对容器资源的控制能力安全
*   Linux内核的能力机制所带来的操作权限安全
*   docker程序（尤其是服务端）本身的抗攻击性
*   其他安全增强机制（包括APPArmor、SELinux等）对容器安全的影响
*   通过第三方工具（包括docker bench工具）对docker环境的安全性的评估

<!--more-->

## 2\. 风险的分析

### 2.1 命名空间隔离的安全

当我们在运行docker run的时候，docker会对针对容器创建一个隔离的命名空间，通过这个命名空间，将容器之间的进程和网络进行隔离，这就意味着容器不能独立的访问其他容器的接口或者套接字。

我们知道，所有容器的网络都是通过docker0进行桥接，当然，如果宿主机上面做一些特殊的配置，可以实现 container1->宿主机->container2 网络的交互方式。

那么，命名空间的架构设计本身，是否是足够安全呢？

其实，命名空间出现的历史很长了，从Linux内核2.6.15的版本（大概是2008年）就已经开始引用了命名空间，但是实际上，“命名空间”这个含义要更早，最开始是从2005年开始提出来的，所以设计和实现足够成熟。

当然，和虚拟机相比，命名空间并不是绝对。因为命名空间，实际上是“假隔离”，虚拟机是“真隔离”。运行在容器内的应用，会直接访问宿主机上面的内核和部分文件。所以，归根结底，我们应该保证的是镜像是足够安全的，只有镜像是安全的，才能保证我们能够在Linux运行安全可信的服务。

### 2.2 控制组资源控制的安全

CGroups有一个重要的作用就是资源审计和资源限制

当我们再运行docker run的时候，docker会通过Linux的相关调用，在后台创建一个控制组，用来控制容器对宿主机的资源消耗，比如控制容器使用内存、CPU等。

控制组有很多重要的作用。比如确保每个容器能够合理的使用共享资源，最重要的是可以通过控制组限制资源的使用，这一点在防止DDoS的时候尤其重要。

**_对于PaaS、容器云这样的容器服务平台，运行着成千上万个容器的实例，如果一旦某个容器被DDoS攻击，那么就会控制组的作用就显现出来，这样可以防止单个容器抢占过多资源，导致整个服务平台出现雪崩！_**

### 2.3 内核能力机制

传统的Unix系统对进程的权限其实只有root权限和非root权限两种粗粒度。

后来，随着Linux内核的升级，开始对权限的粒度越来越灵活，例如，可以给用户分配某个文件的修改权限、可以给某个用户操作某个进程的权限等等。

默认情况下，docker在运行容器的时候，只使用Linux内核的一部分能力，而且，容器的一些能力往往也是由宿主机上面的一些服务进行支持，比如网络的管理等。所以docker其实并不需要获取真正的“root权限”，此外，容器还能禁用一些不必要的权限，比如：

*   禁止任何文件挂载操作（挂载实际上是宿主机，而不是容器本身）；
*   禁止访问宿主机上面的套接字；
*   禁止访问一些文件系统的操作，比如创建新设备；
*   禁止模板加载

**_所以，及时攻击者入侵到容器内部，在容器内部获取了root权限，也并不是真正的宿主机上面的“root权限”，能进行的破坏也是有限的。_**

### 2.4 Docker服务端的防护

使用docker最核心的就是docker服务器了。由于现在启动docker服务器需要使用root权限，所以服务端的权限显得尤其重要。

首先，我们应该确保运行docker的用户是可信的人。由于容器的内部一般都是root权限，如果某个恶意的用户，将宿主机上面的/目录映射到容器内部，那么容器理论上就会有修改根目录下面的权限。因此，在创建容器的时候，我们应该详细检查运行的参数。

尽量将容器映射到非root权限的用户目录下面，这样，可以有效减轻容器和宿主机上面因为权限而导致的安全隐患。

允许docker服务端在非root权限下运行，利用安全可靠的子进程限制特殊权限的操作。比如，这些子进程只能负责文件管理、配置等操作。

### 2.5 更多安全特性的使用

除了docker能力机制之外，我们可以使用一些安全软件增加docker的安全性。比如APParmor等。

docker默认只启用了能力机制。用户还可以启用更多的方案加强docker安全：

*   在内核中启用GRSEC和PAX，这样可以增加更多编译和运行的检查；并且通过地址随机化机制避免恶意探测。启动该特性不需要docker进行任何配置。[社区最佳实践：基于PaX/Grsecurity & STIG & Sheild针对es的Docker场景化加固](https://hardenedlinux.github.io/system-security/2015/09/06/hardening-es-in-docker-with-grsec.html)
*   使用一些增强安全特性的容器模板，比如带APParmor的模板和Redhat带SELinux的策略的模板。这些模板提供了额外的安全特性。[AppArmor security profiles for Docker]("https://docs.docker.com/engine/security/apparmor)
*   用户可以自定义更加严格的访问控制机制来制定安全策略。

此外，将宿主机的文件挂载到容器内部的时候，可以通过设置一些只读（read-only）权限来避免容器对宿主机文件系统的破坏，特别是一些系统运行状态的目录，包括/proc/sys、/proc/irq、/proc/bus等等。

### 2.6 使用第三方检测工具

前面说了很多加强docker安全性的方式，但是注意去检查，会比较繁琐。幸亏现在有一些自动化的检测工具，比较出名的就是docker bench和Clair。

#### 2.6.1 docker bench

docker bench其实是一个docker的镜像，仓库地址：[https://hub.docker.com/r/docker/docker-bench-security/](https://hub.docker.com/r/docker/docker-bench-security/) 通过运行docker bench，可以对docker的一些配置做自动化安全检测。检测的标准是CIS Docker，检测项包括主机配置、Docker引擎、配置文件权限、镜像管理、容器运行时环境、安全项等6大方面。

```bash
$ docker run -it --net host --pid host --userns host --cap-add audit_control \
    -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
    -v /var/lib:/var/lib \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v /usr/lib/systemd:/usr/lib/systemd \
    -v /etc:/etc --label docker_bench_security \
    docker/docker-bench-security
```

![Alt text](/uploads/2018/09/docker-bench-1.png) ![Alt text](/uploads/2018/09/docker-bench-2-1024x650.png)

在输出的结果中，会给出响应的提示信息，然后用户可以根据对应的提示，进行一些配置的更改等操作。一般是避免出现WARN或以上的问题。

## 3\. 总结

docker其实自身携带的一些基本的抵御安全风险的机制，配合APParmor等安全机制，可以让docker容器更加安全。任何技术层面的实现，都需要合理的使用才能等到巩固，特别是生产环境，可能遭遇很多位置的安全风险，所以需要配合完善的监控系统来加强管理。

Docker使用的时候需要注意：

*   容器自身携带的隔离，并不是很完善，需要加强对容器的安全审查。
*   尽量使用官方的镜像，降低安全风险。
*   采用专门的服务器用来管理docker服务，加强对容器的监控机制。
*   随着容器的大规模使用，甚至构成容器集群的时候，需要考虑容器网络上必备的安全防护，比如DDoS攻击等。