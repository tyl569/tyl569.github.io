---
title: 基于以太坊创建私有链进行挖矿、交易
tags:
  - blockchain
  - 以太坊
  - 比特币
id: '206'
categories:
  - - Linux
  - - 以太坊
comments: false
date: 2018-01-23 14:18:36
---

要说2018年什么最火，无疑就是区块链。比特币的疯狂上涨，每个比特币超过了1万美金。随之而来的就是区块链的技术。 以太坊（Ethereum）并不是一个机构，而是一款能够在区块链上实现智能合约、开源的底层系统。本文主要是通过以太坊，创建私有链，实现挖矿和交易。

#### 安装golang

##### 克隆项目

```bash
$ git clone https://github.com/golang/go.git
```

##### 安装go 1.4

golang 是自编译，所以如果安装版本 >=1.5 需要先编译1.4版本，然后再安装其他版本

```bash
$ cp -r go/ $HOME/go1.4 #复制一份文件夹，用于编译1.4版本
$ cd $HOME/go1.4
$ git checkout release-branch.go1.4
$ cd src
$ ./make.bash # 进行编译
```

编译之后，开始安装go 1.9版本

```bash
$ cd $HOME/install/go
$ git checkout release-branch.go1.9
$ cd src/
$ ./all.bash #安装1.9版本
##### Building Go bootstrap tool.
cmd/dist

##### Building Go toolchain using /home/test/go1.4.
bootstrap/cmd/internal/dwarf
bootstrap/cmd/internal/objabi
bootstrap/cmd/internal/src
bootstrap/cmd/internal/sys
bootstrap/cmd/internal/obj
bootstrap/cmd/internal/obj/arm
bootstrap/cmd/internal/obj/arm64
bootstrap/cmd/internal/obj/mips
bootstrap/cmd/internal/obj/ppc64
bootstrap/cmd/internal/obj/s390x
... ##各种编译安装信息
##### API check
Go version is "go1.9.2", ignoring -next /home/test/install/go/api/next.txt

ALL TESTS PASSED

---
Installed Go for linux/amd64 in /home/test/install/go
Installed commands in /home/test/install/go/bin
*** You need to add /home/test/install/go/bin to your PATH.
```

配置环境变量

```bash
$ export PATH=$PATH:/home/test/install/go/bin
```

查看安装版本

```bash
$ go version
go version go1.9.2 linux/amd64
```

##### 克隆go-ethereum

```bash
$ git clone https://github.com/ethereum/go-ethereum.git
```

##### 安装以太坊

```bash
$ make geth

build/env.sh go run build/ci.go install ./cmd/geth
>>> /home/test/install/go/bin/go install -ldflags -X main.gitCommit=5d4267911a7791bfa60f275a97347372fbf0ce99 -v ./cmd/geth
github.com/ethereum/go-ethereum/common/hexutil
github.com/ethereum/go-ethereum/crypto/sha3
github.com/ethereum/go-ethereum/common
...
github.com/ethereum/go-ethereum/vendor/github.com/gizak/termui
github.com/ethereum/go-ethereum/vendor/github.com/naoina/go-stringutil
github.com/ethereum/go-ethereum/vendor/github.com/naoina/toml/ast
github.com/ethereum/go-ethereum/vendor/github.com/naoina/toml
github.com/ethereum/go-ethereum/cmd/geth
Done building.
Run "/home/test/install/go-ethereum/build/bin/geth" to launch geth.
```

##### 创建连接

```bash
$ ln -s  /home/test/install/go-ethereum/build/bin/geth /usr/local/bin/geth
```

#### 创建私有链

##### 创建创世区块

```json
{
    "config": {
        "chainId": 15,
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
    },
    "coinbase" : "0x0000000000000000000000000000000000000000",
    "difficulty" : "0x40000",
    "extraData" : "",
    "gasLimit" : "0xffffffff",
    "nonce" : "0x0000000000000042",
    "mixhash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
    "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
    "timestamp" : "0x00",
    "alloc": { }
}
```

```bash
$ mkdir my_chain
$ cd my_chain
$ vim genesis.json # json文件的内容是上面的json字符串
```

##### 创建创世节点，并且初始化数据

```bash
$ geth --datadir data00 init genesis.json
```

`data00`就是用来保存创世节点的数据

##### 启动节点，指定networkid

```bash
$ geth --datadir ./data00 --networkid 5201314 console #使用console 支持命令行模式
```

![](/uploads/2018/01/00.png)

##### 创建节点的账号

```bash
# 在console命令模式下
> personal.newAccount("123")
"0x9ff8676095e5999bf82eafeab98192e33ad74364"
```

##### 开始进行挖矿

```bash
> miner.start()
INFO [01-2300:14:37] Updated mining threads                   threads=0
INFO [01-2300:14:37] Transaction pool price threshold updated price=18000000000
INFO [01-2300:14:37] Etherbase automatically configured       address=0x9FF8676095e5999bf82EafEaB98192E33ad74364
null
> INFO [01-2300:14:37] Starting mining operation
INFO [01-2300:14:37] Commit new mining work                   number=1 txs=0 uncles=0 elapsed=146.511µs
INFO [01-2300:14:43] Generating DAG in progress               epoch=0 percentage=0 elapsed=4.529s
INFO [01-2300:14:48] Generating DAG in progress               epoch=0 percentage=1 elapsed=8.913s
INFO [01-2300:14:52] Generating DAG in progress               epoch=0 percentage=2 elapsed=13.169s
INFO [01-2300:14:56] Generating DAG in progress               epoch=0 percentage=3 elapsed=17.298s
INFO [01-2300:15:01] Generating DAG in progress               epoch=0 percentage=4 elapsed=21.964s
...
```

等到percentage加载到100的时候就开始进行挖矿

![](/uploads/2018/01/%E6%8C%96%E7%9F%BF.png)

##### 结束挖矿

```bash
> miner.stop()
```

##### 查看挖矿的金额

```bash
> eth.getBalance(eth.accounts[0])
5000000000000000000
```

##### 新开一个窗口，创建第二个节点

```bash
$ geth --datadir data01 init genesis.json
```

##### 运行第二个节点

networkid需要和第一个账号相同

```bash
$ geth --datadir data01 --networkid 5201314 --ipcdisable --port 61910 --rpcport 8200 console #使用命令行模式
```

##### 创建账号

```bash
> personal.newAccount("456")
"0xfc350d17b0fb92eeb0a3bab80116c27d9f7e40d1"
```

##### 开始进行挖矿

```bash
> miner.start()
```

##### 结束挖矿

```bash
> miner.stop()
```

#### 用户交易

##### 回到第一个节点

查看节点信息

```bash
> admin.nodeInfo.enode
"enode://49e538b3f090a04e97f56a7fd1e6223c29599535d5e93010349147dee334b690744504f057ae11adb2804baada222375a56398ef42be536c595c6197a4a7cb2d@[::]:30303"
```

##### 切换到第二个节点窗口

建立联系, 添加第一个节点enode

```bash
 admin.addPeer("enode://49e538b3f090a04e97f56a7fd1e6223c29599535d5e93010349147dee334b690744504f057ae11adb2804baada222375a56398ef42be536c595c6197a4a7cb2d@[::]:30303")
true
```

##### 切换到第一个控制台

查看建立的联系数量

```bash
> net.peerCount
1
```

peerCount=1，说明已经建立了联系

##### 开始进行交易

切换到一个控制台，交易之前，需要先解锁账号才行

```bash
> personal.unlockAccount(eth.accounts[0], "123")
true
```

返回true说明已经解锁成功

```bash
> eth.sendTransaction({from: "0x9ff8676095e5999bf82eafeab98192e33ad74364", to: "0xfc350d17b0fb92eeb0a3bab80116c27d9f7e40d1", value: web3.toWei(1, "ether")})
```

to和form分别是接受和发送的账号，也就是personal.listAccounts里面的账号

查看确认下交易信息

```bash
> eth.pendingTransactions
[{
    blockHash: null,
    blockNumber: null,
    from: "0x9ff8676095e5999bf82eafeab98192e33ad74364",
    gas: 90000,
    gasPrice: 18000000000,
    hash: "0x6146513432b27b6a27f54b64fcf0a30dc90290452dfd25e282a05aaf423f4afa",
    input: "0x",
    nonce: 0,
    r: "0x77644ff132f800da9b2d8133c796916f956061a491b55e3cbbd0d710f5157199",
    s: "0x9dbe268e152e5fc15421efa5b00e5d4a601ef4cb2655b577901fd96c5d3c959",
    to: "0xfc350d17b0fb92eeb0a3bab80116c27d9f7e40d1",
    transactionIndex: 0,
    v: "0x42",
    value: 1000000000000000000
}]
```

开始进行挖矿，使交易生效

```bash
> miner.start()
```

##### 确认交易是否成功

在一个和第二个控制台分别运行命令，确认是否交易成功

```bash
> eth.getBalance(eth.accounts[0])
```

#### 总结

##### 挖矿对电脑要求比较高，我使用阿里云的服务器，1核1G基本的配置，两个节点同时挖矿，经常出现CPU吃满的情况。

![](/uploads/2018/01/CPU%E5%90%83%E6%BB%A1.png)

![](/uploads/2018/01/io%E8%AF%BB%E5%86%99.png)

##### 还有一个比较奇怪的，当发起交易的时候，一定要进行挖矿操作，才能使交易生效。

#### 参考文献

[blockchain随笔](http://www.cnblogs.com/zl03jsj/category/997608.html)