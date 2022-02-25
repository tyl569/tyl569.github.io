---
layout: post
title: docker之运行golang
date: 2018-09-02
tags: ["Docker","Linux"]
---

众所周知，docker解决了编程的痛点问题--运行环境，所以我先走基本上尽量都使用docker运行。这样做，首先就是让我不必关心配置复杂的运行环境，另外也可以让我更加熟练的使用docker。

    //go-sample.go
    package main
    import "fmt"
    func main() {
        fmt.Println("hello world");
    }

## Golang:onbuild

现在关于go的docker镜像也发布了很多个版本，我们首先介绍一下`golang:onbuild`以及如何使用。
`golang:onbuild`是go语言官方发布的一款很小的镜像(只有几KB大小)，目的是为了让我们可以编译go文件，并且运行。使用的方式很简单，只需要创建一个Dockerfile，然后在首行加上`FROM golang:onbuild`。

    -rw-r--r--@ 1 feilong wheel 20 9 2 00:23 Dockerfile
    -rw-r--r-- 1 feilong wheel 72 9 2 00:03 go-sample.go
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$ cat Dockerfile
    FROM golang:onbuild
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$ docker build -t golang_onbuild .
    Sending build context to Docker daemon  3.072kB
    Step 1/1 : FROM golang:onbuild
    onbuild: Pulling from library/golang
    ad74af05f5a2: Pull complete
    2b032b8bbe8b: Pull complete
    a9a5b35f6ead: Pull complete
    25d9840c55bc: Pull complete
    d792ec7d64a3: Pull complete
    be556a93c22e: Pull complete
    3a5fce283a1e: Pull complete
    0621865a0c2e: Pull complete
    Digest: sha256:c0ec19d49014d604e4f62266afd490016b11ceec103f0b7ef44875801ef93f36
    Status: Downloaded newer image for golang:onbuild
    # Executing 3 build triggers
     ---> Running in 109c7a7ebeb5
    + exec go get -v -d
    Removing intermediate container 109c7a7ebeb5
     ---> Running in c0dfd28de95e
    + exec go install -v
    app
    Removing intermediate container c0dfd28de95e
     ---> 820e315d7160
    Successfully built 820e315d7160
    Successfully tagged golang_onbuild:latest
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$ docker run -it --rm --name go_onbuild golang_onbuild
    + exec app
    hello world

我们根据docker:onbuild的Dockerfile文件具体分析一个整个编译的过程(以1.3.1版本为例)

    FROM golang:1.3.1

    RUN mkdir -p /go/src/app
    WORKDIR /go/src/app

    # this will ideally be built by the ONBUILD below ;)
    CMD ["go-wrapper", "run"]

    ONBUILD COPY . /go/src/app
    ONBUILD RUN go-wrapper download
    ONBUILD RUN go-wrapper install

从Dockerfile和build过程可以看出，在进行build的时候，经历了三次触发器:

*   首先，将当前目录拷贝到`. /go/src/app`
*   下载对应的依赖包
*   编译安装

编译之后，golang:onbuild镜像默认包含了一个CMD ["app"] 命令，用来执行编译后的go文件。

我们通过实际run一个容器验证一下：

    feilongdeMBP:go feilong$ docker run -it --rm --name golang_onbuild golang_onbuild
    + exec app
    hello world

##  Golang:latest

相比较golang:onbuild的便利性，golang:latest就变得很灵活了，需要我们手动编译go文件，然后手动执行编译后的文件。因为毕竟电脑并不知道你具体想要编译的顺序，以及你要想要执行的编译文件。运行过程如下：

    feilongdeMBP:go feilong$ ll
    total 16
    -rw-r--r--@ 1 feilong  wheel  133  9  2 00:04 Dockerfile
    -rw-r--r--  1 feilong  wheel   72  9  2 00:03 go-sample.go
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$ cat Dockerfile
    FROM golang:latest

    RUN mkdir -p /go/src/app
    WORKDIR /go/src/app

    COPY . /go/src/app
    RUN go build -o app .
    CMD [ "/go/src/app/app" ]
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$
    feilongdeMBP:go feilong$ docker build -t go_go .
    Sending build context to Docker daemon 3.072kB
    Step 1/6 : FROM golang:latest
     ---> 7e9ac7032e33
    Step 2/6 : RUN mkdir -p /go/src/app
     ---> Running in b5d3f63578ed
    Removing intermediate container b5d3f63578ed
     ---> 95c2beb49121
    Step 3/6 : WORKDIR /go/src/app
     ---> Running in 3011d74944c9
    Removing intermediate container 3011d74944c9
     ---> 82d6a45aa3e3
    Step 4/6 : COPY . /go/src/app
     ---> 475b2bdd5769
    Step 5/6 : RUN go build -o app .
     ---> Running in 5802ac0c98b4
    Removing intermediate container 5802ac0c98b4
     ---> 7a019370f09d
    Step 6/6 : CMD [ "/go/src/app/app" ]
     ---> Running in a3f6ad19d2ef
    Removing intermediate container a3f6ad19d2ef
     ---> 635417bdcda8
    Successfully built 635417bdcda8
    Successfully tagged go_go:latest

run一个容器，查看运行效果

    feilongdeMBP:go feilong$ docker run -it --rm --name go go_go
    hello world

## 总结

golang:onbuild和golang:lastest各有利弊，前者更加简单，能够更加简明扼要的告诉我们运行过程，而后者更加灵活，将更多的操作命令交给了开发人员。

## 参考文献

*   [https://time-track.cn/build-minimal-go-image.html](https://time-track.cn/build-minimal-go-image.html)