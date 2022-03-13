---
title: docker之命名空间
tags:
  - docker
id: '332'
categories:
  - - Docker
  - - Linux
date: 2018-09-10 23:19:32
---

## 1\. 基本架构

docker目前采用了标准的C/S架构。客户端和服务端既可以运行在一个机器上，又可以通过socket或者restful API来进行通信。

### 1.1 服务端

docker服务端一般都是在宿主机上，来接受客户端的命令。docker默认使用套接字的方式，但是也是允许使用tcp进行端口的监听，可以使用docker daemon -H IP:PORT的方式进行监听。

<!--more-->

### 1.2 客户端

docker的客户端主要作用是向服务端发送操作的指令。客户端默认也是采用套接字的方式，向本地的docker服务端发送命令。当然，客户端也是可以使用tcp的方式进行发送指令，使用docker -H tcp://IP:PORT，用来指定接收命令的docker服务端。

## 2\. 命名空间

大家在平时使用Linux或者macos的时候，我们并没有拆分多个环境的需求。但是在服务器上面，加入一台服务器运行多个进程，进程之间是相互影响的，比如共享内存，操作相同的文件。我们其实更希望能够将这些进程分离开，这样情况下，如果服务受到攻击，不会影响其他的服务。

![Alt text](/uploads/2018/09/屏幕快照-2018-09-04-下午10.59.44.png)

docker目前主要有6命名空间的隔离方式

### 2.1 进程空间隔离

进程在操作系统中是一个很重要的概念，也就是大家认为的正在运行中的程序。

```bash
feilongdeMBP:~ feilong$ ps -ef
UID PID PPID C STIME TTY TIME CMD
0 1 0 0 9:31下午 ?? 0:10.07 /sbin/launchd
0 44 1 0 9:31下午 ?? 0:00.65 /usr/sbin/syslogd
0 45 1 0 9:31下午 ?? 0:01.37 /usr/libexec/UserEventAgent (System)
0 48 1 0 9:31下午 ?? 0:00.25 /System/Library/PrivateFrameworks/Uninstall.framework/Resources/uninstalld
0 49 1 0 9:31下午 ?? 0:02.57 /usr/libexec/kextd
0 50 1 0 9:31下午 ?? 0:02.40 /System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/FSEvents.framework/Versions/A/Support/fseventsd
0 52 1 0 9:31下午 ?? 0:00.16 /System/Library/PrivateFrameworks/MediaRemote.framework/Support/mediaremoted
55 55 1 0 9:31下午 ?? 0:00.38 /System/Library/CoreServices/appleeventsd --server
0 56 1 0 9:31下午 ?? 0:00.75 /usr/sbin/systemstats --daemon
```

可见当前系统运行了很多“程序”。

我们现在新建一个容器，然后进入容器看下，docker容器里面的进程列表

```bash
feilongdeMBP:~ feilong$ docker run -it --rm --name test busybox
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    6 root      0:00 ps -ef
```

对比很明显，容器内部只有很少的几个正在运行的进程。

我们新建一个窗口，然后看下宿主机上面和docker相关的进程

```bash
localhost:~ feilong$ ps -ef  grep docker
    0    82     1   0  9:31下午 ??         0:00.02 /Library/PrivilegedHelperTools/com.docker.vmnetd
  501   918   879   0 10:26下午 ??         0:00.14 /Applications/Docker.app/Contents/MacOS/com.docker.supervisor -watchdog fd:0
  501   920   918   0 10:26下午 ??         0:03.32 com.docker.osxfs serve --address fd:3 --connect vms/0/connect --control fd:4 --log-destination asl
  501   921   918   0 10:26下午 ??         0:00.73 com.docker.vpnkit --ethernet fd:3 --port fd:4 --diagnostics fd:5 --pcap fd:6 --vsock-path vms/0/connect --host-names host.docker.internal,docker.for.mac.host.internal,docker.for.mac.localhost --gateway-names gateway.docker.internal,docker.for.mac.gateway.internal,docker.for.mac.http.internal --vm-names docker-for-desktop --listen-backlog 32 --mtu 1500 --allowed-bind-addresses 0.0.0.0 --http /Users/feilong/Library/Group Containers/group.com.docker/http_proxy.json --dhcp /Users/feilong/Library/Group Containers/group.com.docker/dhcp.json --port-max-idle-time 300 --max-connections 2000 --gateway-ip 192.168.65.1 --host-ip 192.168.65.2 --lowest-ip 192.168.65.3 --highest-ip 192.168.65.254 --log-destination asl --udpv4-forwards 123:127.0.0.1:59434 --gc-compact-interval 1800
  501   922   918   0 10:26下午 ??         0:01.17 com.docker.driver.amd64-linux -addr fd:3 -debug
  501   928   922   0 10:26下午 ??         2:40.08 com.docker.hyperkit -A -u -F vms/0/hyperkit.pid -c 2 -m 2048M -s 0:0,hostbridge -s 31,lpc -s 1:0,virtio-vpnkit,path=vpnkit.eth.sock,uuid=246fb3f9-3ad5-4683-837a-33ac39f57f25 -U 5a3669ae-b209-443a-a074-312cd32a258a -s 2:0,ahci-hd,/Users/feilong/Library/Containers/com.docker.docker/Data/vms/0/Docker.raw -s 3,virtio-sock,guest_cid=3,path=vms/0,guest_forwards=2376;1525 -s 4,ahci-cd,/Applications/Docker.app/Contents/Resources/linuxkit/docker-for-mac.iso -s 5,ahci-cd,vms/0/config.iso -s 6,virtio-rnd -s 7,virtio-9p,path=vpnkit.port.sock,tag=port -l com1,autopty=vms/0/tty,asl -f bootrom,/Applications/Docker.app/Contents/Resources/uefi/UEFI.fd,,
  501  2074  1102   0 11:21下午 ??         0:00.50 /Applications/Visual Studio Code.app/Contents/Frameworks/Code Helper.app/Contents/MacOS/Code Helper /Users/feilong/.vscode/extensions/peterjausovec.vscode-docker-0.1.0/node_modules/vscode-languageclient/lib/utils/electronForkStart /Users/feilong/.vscode/extensions/peterjausovec.vscode-docker-0.1.0/node_modules/dockerfile-language-server-nodejs/lib/server.js --node-ipc --node-ipc --clientProcessId=1102
  501  2100  1065   0 11:24下午 ttys001    0:00.12 docker run -it --rm --name test busybox
  501  2086  2083   0 11:21下午 ttys002    0:00.19 docker exec -it 910aa64a312b3a884f4efb059e47ee601bbd3ba3d62f4c92abd4120cff770828 /bin/sh
  501  2090  2087   0 11:21下午 ttys003    0:00.12 docker exec -it 73f8fbcc50651fd4fea9fe0be7fe4066ea78efd7e9b2438fe657a3e7725e7903 /bin/sh
  501  2115  2111   0 11:27下午 ttys004    0:00.00 grep docker
```

在进程列表中，我们没有看到容器内部运行的进程，说明相对于容器的“外部”，容器“内部”的进程是隔离的。但是我们也可以发现，刚刚创建的名字为test的容器，实质上就是宿主机上面的一个PID为2090的进程。

所以，我们可以理解docker的进程树是这个状态：

![Alt text](/uploads/2018/09/屏幕快照-2018-09-04-下午11.43.31.png)

### 2.2 网络空间隔离

容器其实不能完全和宿主机器隔离网络，要不然的话容器就没办法通过外部进行访问，那么也就没有实际的意义。但是容器之间是网络隔离的，这种隔离的方式，就是通过网络命名空间实现的。

docker有四种不同的网络模式：Host、Container、None和bridge

docker默认的是桥接模式。

docker在创建容器的时候， 不仅会给容器创建IP地址，还会在宿主机上面创建一个虚拟网桥docker0，在运行的时候，将容器和该网桥进行相连。

在默认的情况下，创建容器的时候，都会创建一对虚拟网卡，两个虚拟网卡组成数据通道，一个在容器内部，另外一个加入到docker0的网桥中。

打开两个窗口，分别创建redis和redis2容器

```bash
[root@izj6c9b96ia369l2i47yq3z feilong]# docker run -it --rm --name redis  -p 6379:6379 redis:latest /bin/bash
root@d89535b59b0b:/data#
[root@izj6c9b96ia369l2i47yq3z feilong]# docker run -it --rm --name redis2 -p 6378:6379 redis:latest /bin/bash
root@7736850135af:/data#
```

```bash
打开第三个窗口，查看网桥的状态
<pre class="prettyprint">[feilong@izj6c9b96ia369l2i47yq3z ~]$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024219a15f9d       no              veth8331b03
                                                        vethc5f3cb9
```

![Alt text](/uploads/2018/09/屏幕快照-2018-09-06-下午8.43.46.png)

docker0 会为每一个容器分配一个新的 IP 地址并将 docker0 的 IP 地址设置为默认的网关。网桥 docker0 通过 iptables 中的配置与宿主机器上的网卡相连，所有符合条件的请求都会通过 iptables 转发到 docker0 并由网桥分发给对应的机器。同时也会在防火墙加上一条新的规则。

```bash
[root@izj6c9b96ia369l2i47yq3z feilong]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-USER  all  --  anywhere             anywhere
DOCKER-ISOLATION-STAGE-1  all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  anywhere             172.17.0.2           tcp dpt:6379
ACCEPT     tcp  --  anywhere             172.17.0.3           tcp dpt:6379
ACCEPT     tcp  --  anywhere             172.17.0.4           tcp dpt:http
......
```

### 2.3 挂载点命名空间

docker已经可以通过命名空间将网络和进程进行隔离。挂载命名空间，允许不同的容器，查看到不同的文件结构，这样，每个命名空间的进程所看到的文件目录彼此被隔离。每个容器内的进程只会更改容器内部的文件目录。

### 2.4 IPC命名空间

容器中的进程交互采用的是Linux中常见的进程间交互方式（Interprocess Communication， IPC），包括信号量、消息队列和内存共享等。IPC命名空间和PID命名空间可以组合使用，同一个IPC命名空间的进程可以彼此可见，允许进行交互，不同空间的进程无法交互。

### 2.5 UTS 命名空间

UTS（Unix time-sharing system）命名空间允许每个容器拥有一个独立的主机名和域名，从而可以虚拟出一个独立的主机名和网络空间的环境，就可以跟网络上的一台独立主机一样。

默认情况下，docker的主机名是容器的id

![Alt text](/uploads/2018/09/WX20180908-004426@2x-1024x293.png)

![Alt text](/uploads/2018/09/WX20180908-004332@2x-1024x55.png)

### 2.6 用户命名空间

每个容器内部都有不同的用户组和组id，也就是说可以在容器内部使用特定的内部用户执行程序，而不是宿主机上的用户。每个容器都有root账号，但是和宿主机都不在一个命名空间。通过使用命名空间隔离，来保证容器内部用户无法操作容器外部的操作权限。

## 3\. 总结

6种命名空间让容器之间松耦合，也让容器与宿主机松偶尔。同时，也保证了安全性。容器内部不能操作其他容器内部的东西，docker的这种命名空间隔离的方式，也比较符合Linux的系统设计。