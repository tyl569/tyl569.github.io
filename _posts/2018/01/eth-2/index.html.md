---
layout: post
title: 基于以太坊实现局域网多节点挖矿
date: 2018-01-25
tags: ["Linux","以太坊","以太坊","区块链","私有链"]
---

上一篇文章简要介绍了本地实现私有链挖矿和转账，现在这篇文章主要实现局域网下实现多个节点实现挖矿

### 前提，已经安装了go-ethereum，如果没有安装请移步[基于以太坊创建私有链进行挖矿、交易](http://feilong.tech/?p=206)

### 机器：Ubuntu(两个节点)，Mac(一个节点)

### 创建创世节点

#### 创建节点json文件

    $ mkdir my_eth2
    $ cd my_eth2
    $ vim genesis.json
    {
      "config": {
            "chainId": 10,
            "homesteadBlock": 0,
            "eip155Block": 0,
            "eip158Block": 0
        },
      "coinbase"   : "0x0000000000000000000000000000000000000000",
      "difficulty" : "0x20000",
      "extraData"  : "",
      "gasLimit"   : "0x2fefd8",
      "nonce"      : "0x0000000000000042",
      "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
      "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
      "timestamp"  : "0x00",
      "alloc"      : {}
    }

#### 生成节点(以下使用节点1代指)

    $ geth --datadir data00 init genesis.json
    WARN [01-25'17:04:25] No etherbase set and no accounts found as default
    INFO [01-25'17:04:25] Allocated cache and file handles         database=/home/ubuntu/my_eth2/data00/geth/chaindata cache=16 handles=16
    INFO [01-25'17:04:25] Writing custom genesis block
    INFO [01-25'17:04:25] Successfully wrote genesis state         database=chaindata                                  hash=5e1fc7...d790e0
    INFO [01-25'17:04:25] Allocated cache and file handles         database=/home/ubuntu/my_eth2/data00/geth/lightchaindata cache=16 handles=16
    INFO [01-25'17:04:25] Writing custom genesis block
    INFO [01-25'17:04:25] Successfully wrote genesis state         database=lightchaindata                                  hash=5e1fc7...d790e0

#### 启动节点1

    $ geth --datadir ./data00 --networkid 5201314 console

#### 创建账号

    > personal.newAccount("123")
    "0x0b514e769e4e1990f8fb0f0f9d876d7f2b9fa5ba"

### 本地创建第二个节点(以下使用节点2代指)

#### 新开窗口，创建节点2

    $ geth --datadir data01 init genesis.json
    WARN [01-25'17:07:33] No etherbase set and no accounts found as default
    INFO [01-25'17:07:33] Allocated cache and file handles         database=/home/ubuntu/my_eth2/data01/geth/chaindata cache=16 handles=16
    INFO [01-25'17:07:33] Writing custom genesis block
    INFO [01-25'17:07:33] Successfully wrote genesis state         database=chaindata                                  hash=5e1fc7...d790e0
    INFO [01-25'17:07:33] Allocated cache and file handles         database=/home/ubuntu/my_eth2/data01/geth/lightchaindata cache=16 handles=16
    INFO [01-25'17:07:33] Writing custom genesis block
    INFO [01-25'17:07:33] Successfully wrote genesis state         database=lightchaindata                                  hash=5e1fc7...d790e0

#### 运行节点2

    $ geth --datadir data01 --networkid 5201314 --ipcdisable --port 61910 --rpcport 8200 console

#### 创建节点2的账号

     > personal.newAccount("123")
    "0x3babf1eeb8d5d29acc4d1f6408529b36b4e6f880"

### 在Mac上创建新节点，以下使用(节点3代指)

`创世节点的json文件要和Ubuntu一致`

#### 初始化节点3

    $ geth --datadir data00 init genesis.json
    WARN [01-25'17:14:10] No etherbase set and no accounts found as default
    INFO [01-25'17:14:10] Allocated cache and file handles         database=/Users/feilong/my_chain2/data00/geth/chaindata cache=16 handles=16
    INFO [01-25'17:14:10] Writing custom genesis block
    INFO [01-25'17:14:10] Successfully wrote genesis state         database=chaindata                                      hash=5e1fc7...d790e0
    INFO [01-25'17:14:10] Allocated cache and file handles         database=/Users/feilong/my_chain2/data00/geth/lightchaindata cache=16 handles=16
    INFO [01-25'17:14:10] Writing custom genesis block
    INFO [01-25'17:14:10] Successfully wrote genesis state         database=lightchaindata                                      hash=5e1fc7...d790e0

#### 运行节点3

    $ geth --datadir data00 --networkid 5201314 --ipcdisable --port 61911 --rpcport 8200 console #使用61911端口，保证networkid一致

#### 创建账号

    > personal.newAccount("123")
    "0xf81b1d6c0e0835790c7e4af8a02301a67e5a0dcb"

### 节点1和节点2建立联系

#### 节点2运行 `> admin.nodeInfo.enode` 查看node信息

    > admin.nodeInfo.enode
    "enode://d5bb9fecc8e997905220b5e8c0db8396880bd5326143614b33f81ead534fc4d8282cbdda620fb81eaea66c359c3acd7d590f64981099b3cc063fddae9ac376d9@192.168.164.210:61910"

#### 节点1添加节点2

    > admin.addPeer("enode://d5bb9fecc8e997905220b5e8c0db8396880bd5326143614b33f81ead534fc4d8282cbdda620fb81eaea66c359c3acd7d590f64981099b3cc063fddae9ac376d9@192.168.164.210:61910")
    true

#### 节点1和节点2运行`> net`

    > net
    {
      listening: true,
      peerCount: 1, #说明添加成功
      version: "5201314",
      getListening: function(callback),
      getPeerCount: function(callback),
      getVersion: function(callback)
    }

### 节点1和节点3建立联系

#### 节点3运行 `> admin.nodeInfo.enode` 查看node信息

    >  admin.nodeInfo.enode
    "enode://34dcd9b7e64b24a25fe25b6e2aab6fc10525a439b2174ad79bd55bbf867f98060f7eef26c83223ae665372afa819ffd5c9c49a039c4e5e9c4e72be35a3b65aa8@192.168.164.210:61911"

#### 节点1添加节点3

    > admin.addPeer("enode://34dcd9b7e64b24a25fe25b6e2aab6fc10525a439b2174ad79bd55bbf867f98060f7eef26c83223ae665372afa819ffd5c9c49a039c4e5e9c4e72be35a3b65aa8@192.168.164.210:61911")
    true

#### 分别查看节点1和节点3链接情况

节点1

    > net
    {
      listening: true,
      peerCount: 2, ##节点1连接两个节点
      version: "5201314",
      getListening: function(callback),
      getPeerCount: function(callback),
      getVersion: function(callback)
    }

节点3

    > net
    {
      listening: true,
      peerCount: 1,
      version: "5201314",
      getListening: function(callback),
      getPeerCount: function(callback),
      getVersion: function(callback)
    }

### 节点挖矿测试

#### 使用任一节点挖矿，然后观察其他两个控制台，发现都会有同步的数据，说明节点2和节点3也是连接的状态（由于电脑性能原因，挖矿的时候需要等percentage到达100之后才会开始）

### 遇到的坑

*   要保证创世节点的json文件一致
*   保证在统一局域网内，使用Telnet命令测试
*   节点2和节点3的端口注意不要重复