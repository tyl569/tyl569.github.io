---
title: 【以太坊】实现智能合约授权与余额转账
tags:
  - token
  - 以太坊
  - 区块链
  - 智能合约
id: '794'
categories:
  - - PHP
date: 2021-06-07 22:06:30
---

上一节实现了智能合约的基本编写以及solidity的基本的函数修饰关键字。这一节开始进行合约的授权和余额的转账。

### 代币授权

首先，代币的授权与余额是一个map，对应key是以太坊钱包的地址，value是对应的授权额度，查询的时间复杂度是O(1)，查询效率会比较高。现在需要实现的是代币余额查询（balanceOf），代币的额度申请（approve）和授权额度查询（allowance）这三个函数。

<!--more-->


```bash

    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);

    // 根据地址获取获取代币金额 
    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }

    // 授权额度申请 
    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowed[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    // 根据 _owner和 _spender查询 _owner给 _spender授权了多少额度 
    function allowance(address _owner, address _spender) public view returns (uint256 remaining) {
        return allowed[_owner][_spender];
    }
```

在approve函数中，emit的关键字含义是触发一个事件授权事件`Approval`。

我们在发布智能合约的第一件事，就是设置代币总金额：

```bash
    constructor() public{
        totalPublic = 1000000000;
        balances[msg.sender] = totalPublic;
    }
```

其中`msg.sender`代表智能合约发布者的以太坊地址，我们把发行量`totalPublic`给予了这个地址。当然这个是可以控制的，只给予一小部分也是允许的。

### 转账函数

一下是智能合约的转账函数：

```bash
   // 转账 
    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(balances[msg.sender] >= _value);
        require(balances[_to] + _value >= balances[_to]);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }
```

转账函数的一些参数校验，一般使用solidity语法中的require函数，首先，验真剩余的额度不能小于转账额度；转账的额度不能是赋值；依次在对收款方和付款方进行额度的增加和减少；然后触发转账事件`Transfer`,把对应的数据写到区块里面。 以上的这个函数，只是合约发布方转账给其他地址。还会有地址之间的代币转账：

```bash
    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        uint256 allowanceValue = allowed[_from][msg.sender];
        require(balances[_from] >= _value && allowanceValue >= _value);
        require(balances[_to] + _value > balances[_to]);
        allowed[_from][msg.sender] -= _value;
        balances[_to] += _value;
        balances[_from] -= _value;
        emit Transfer(_from, _to, _value);
        return true;
    }
```

至此，一份标准的ERC20代币的智能合约就编写完成了，完整代码如下：

```bash
contract FeilongToken {
    string public name="Feilong token coin"; // 代币的名称
    uint8 public decimals = 18;// 精确小数点位数
    string public symbol = "FLTC";//代币符号
    uint public totalPublic;//代币发行量

    mapping (address => uint256) public balances;// 余额map 
    mapping (address => mapping(address =>uint256)) public allowed;// 授权map

    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event Approval(address indexed _owner, address indexed _spender, uint256 _value);

    constructor() public{
        totalPublic = 1000000000;
        balances[msg.sender] = totalPublic;
    }

    // 根据地址获取获取代币金额 
    function balanceOf(address _owner) public view returns (uint256 balance) {
        return balances[_owner];
    }

    // 授权额度申请 
    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowed[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }

    // 根据 _owner和 _spender查询 _owner给 _spender授权了多少额度 
    function allowance(address _owner, address _spender) public view returns (uint256 remaining) {
        return allowed[_owner][_spender];
    }

    // 转账 
    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(balances[msg.sender] >= _value);
        require(balances[_to] + _value >= balances[_to]);
        balances[msg.sender] -= _value;
        balances[_to] += _value;
        emit Transfer(msg.sender, _to, _value);
        return true;
    }

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        uint256 allowanceValue = allowed[_from][msg.sender];
        require(balances[_from] >= _value && allowanceValue >= _value);
        require(balances[_to] + _value > balances[_to]);
        allowed[_from][msg.sender] -= _value;
        balances[_to] += _value;
        balances[_from] -= _value;
        emit Transfer(_from, _to, _value);
        return true;
    }

}
```

### 合约的代码安全

由于智能合约是代码编写的，所以就可能存在代码的漏洞。最著名的合约的漏洞是2018年的“美链BEC合约漏洞事件”。该事件的后果就是“BEC”代币价值归零，以下是漏洞代码：

```bash
    function batchTransfer(address[] _receivers, uint256 _value) public whenNotPaused returns (bool) {
        uint cnt = _receivers.length;
        uint256 amount = uint256(cnt) * _value;
        require(cnt > 0 && cnt <= 20);
        require(_value > 0 && balances[msg.sender] >= amount);

        balances[msg.sender] = balances[msg.sender].sub(amount);
        for (uint i = 0; i < cnt; i++) {
            balances[_receivers[i]] = balances[_receivers[i]].add(_value);
            Transfer(msg,sender, _receivers[i], _value);
        }
        return true;
    }
```

batchTransfer是一个批量函数，可以实现批量转账，但是代码`uint256 amount = uint256(cnt) * _value`存在风险，加入`_value`是一个在uint256范围内，但是乘以cnt得到的`amount`超过了uint256而造成溢出，就会变成一个比较小的数字，当执行代码`require(_value > 0 && balances[msg.sender] >= amount);`也校验通过了，在循环执行转账操作的时候就会出现扣除很少的`amount`，而收款方获得了很大的`_value`，造成了资产的被盗。

解决这种溢出，可以通过除法进行验证，比如：

```bash
    function batchTransfer(address[] _receivers, uint256 _value) public whenNotPaused returns (bool) {
        uint cnt = _receivers.length;
        uint256 amount = uint256(cnt) * _value;
        require(_value > 0 && amount / _value == cnt);
        require(cnt > 0 && cnt <= 20);
        require(_value > 0 && balances[msg.sender] >= amount);

        balances[msg.sender] = balances[msg.sender].sub(amount);
        for (uint i = 0; i < cnt; i++) {
            balances[_receivers[i]] = balances[_receivers[i]].add(_value);
            Transfer(msg,sender, _receivers[i], _value);
        }
        return true;
    }
```

以上只是合约安全的一个简单的例子。在实际开发中，涉及到复杂的运算，一定要谨慎编写代码，以免出现代码漏洞。

本文链接：[https://feilong.tech/approve\_transfer/](https://feilong.tech/approve_transfer/)