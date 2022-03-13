---
title: 【以太坊】编译智能合约
tags:
  - 以太坊
  - 区块链
id: '764'
categories:
  - - Go
  - - PHP
  - - 以太坊
date: 2021-05-24 22:22:19
---

#### 智能合约

说到以太坊开发，就肯定绕不开智能合约。

智能合约，其实是一种协议，就相当于是一种规则，他规定了交易、转账等。智能合约也可以理解成是“一段代码”，开发在通过执行“这段代码”，获得一个结果，这个结果可能是转账结果，或者其他等等。

在开发以太坊的时候，开发者需要先编写智能合约，然后将智能合约部署到对应的以太坊节点，以太坊被部署到不同的服务器上，节点共同维护以太坊公链，调用者通过调用以太坊接口，访问智能合约，获得对应的结果。


![](/uploads/2021/05/企业微信20210524-220210.png)

<!--more-->

#### remix

以太坊也给开发者准备了响应的开发工具————remix，线上地址：[remix-online](https://remix.ethereum.org/#optimize=false&runs=200&evmVersion=null "remix-online")，同时也提供了IDE开发工具，[remix-desktop](https://github.com/ethereum/remix-desktop "remix-desktop")

![](/uploads/2021/05/企业微信20210524-221218.png)

##### 使用方式

和普通的IDE工具一样，remix也支持语法高亮和代码提示，以及报错信息

![](/uploads/2021/05/企业微信20210524-222130.png)

开启自动编译之后，就能实时的预览编辑结果，方便我们及时更正语法错误。

##### 简单的智能合约

我们编写一个简单的加法智能合约

```solidity
contract Test {
    function add(uint8 arg1, uint8 arg2) public pure returns (uint8) {
        return arg1 + arg2;
    }
}
```

![](/uploads/2021/05/企业微信20210524-221218.png)

一个简单的智能合约就实现了。

本文地址： [https://feilong.tech/2021/05/24/eth-contract-demo/](https://feilong.tech/2021/05/24/eth-contract-demo/)