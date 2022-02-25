---
layout: post
title: 【以太坊】实现ERC20代币智能合约
date: 2021-05-30
tags: ["Go","以太坊","以太坊","区块链","智能合约"]
---

以太坊发布的大多数代币的智能合约，都是参照了"ERC20"的标准协议，本节也主要是根据"ERC20"开发一份代币的智能合约，根据上一节的步骤，提前创建一个FeilongToken.sol的文件

#### 创建智能合约

solidity 其实和普通的编程语言有异曲同工之妙，一下的我定义的一些成员变量，我们定义了一些基本的信息，比如代币的名称、代币单位精确的小数点、代币的符号以及发行量。

    contract FeilongToken {
        string public name="Feilong token coin"; // 代币的名称
        uint8 public decimals = 18;// 精确小数点位数
        string public symbol = "FLTC";//代币符号
        uint public totalPublic = 100;//代币发行量
    }

#### 构造函数

solidity也支持构造函数的使用

    contract FeilongToken {
        string public name="Feilong token coin"; // 代币的名称
        uint8 public decimals = 18;// 精确小数点位数
        string public symbol = "FLTC";//代币符号
        uint public totalPublic;//代币发行量

        constructor() public {
            totalPublic = 100;
        }
    }

#### solidity 关键字

下面给大家介绍一些常用的关键字

> ###### public 可以修饰变量和函数，表示合约内部和合约外部都能够进行调用和访问。和其他编程语言功能相同，如果不设置关键字，默认是public类型
> 
> ###### private 可以修饰变量和函数，表示只能当前合约内部能够进行调用和访问，不能被合约外部和子类进行调用和访问。
> 
> ###### external 只能用来修饰函数，只能被合约外部的代码进行调用和访问，不能被当前合约内部的代码和子类进行调用和访问。
> 
> ###### internal 可以修饰变量和函数，但是只能被当前的合约或者子类进行调用和访问，不能被外部进行调用和访问
> 
> ###### view 只能用来修饰函数，函数内部可以对外部变量进行读取操作，但是不能修改
> 
> ###### pure 只能用来修饰函数，函数内部不能对外部变量进行读取和修改惭怍，只能对传参进行读写操作

这几个关键字，不光是约束了外部对自己的调用，view和pure也约束了自己对外部变量的操作限制。

    contract FeilongToken {
        string public name="Feilong token coin"; // 代币的名称
        uint8 public decimals = 18;// 精确小数点位数
        string public symbol = "FLTC";//代币符号
        uint public totalPublic;//代币发行量

        constructor() public{
            totalPublic = 100;
        }

        function getHalfPublic() external view returns (uint half) {
            return totalPublic/2;
        }
        function internalFunc() internal view returns (uint half) {
            return totalPublic/2;
        }
    }

    contract Parent {
        uint64 age = 30;
        address public addr = 0x753B5C00b357b1536aE206C3319582A5A00b9c02;

        function func() public view {
            FeilongToken m = FeilongToken(addr);
            m.getHalfPublic(); // 可以访问
            m.internalFunc(); // 报错，因为internal类型，外部不能进行访问
        }

            function publicFunc() public {
            age += 16;
        }

        function privateFunc() private returns (uint64 ret) {
            uint64 t = internalFunc();
            age = t / 2;
            return age;
        }

        function internalFunc() internal returns (uint64 ret) {
            age *= 2;
            return ret;
        }

        function viewFunc(uint64 arg1) public view returns (uint64 ret) {
            arg1 = arg1 + age + 9; // 这里不会报错，因为view是允许访问函数外部变量的
            age += 7; // 这里会报错，因为是view类型，不能修改外部的变量
            uint64 d = age + arg1;  // 这里不会报错，因为view是允许访问函数外部变量的
            uint64 c = d/2;
            return c;
        }

        function pureFunc(uint64 arg1) public pure returns (uint64 ret) {
            arg1 += 9;
            uint64 d = age + arg1; // 这里会报错，因为pure不允许读取函数外部的变量
            return arg1;
        }
    }

    contract child is Parent {
        function usePrivateFunc() public returns (uint64 ret) {
            uint64 v = privateFunc(); // 这里进行报错，因为private的函数不能被继承
        }

        function useInternalFunc() public returns (uint64 ret) {
            uint64 v = internalFunc();// 这里不会报错，因为internal的函数可以被继承
            return v;
        }
    }

#### 总结

solidity的语法其实和其他编程语言类似，但是有所不同。我们一般的语言，大多数都是限制了外部对自己的调用权限控制，solidity也限定了函数本身对外部的操作。比如view和pure，限制了自己对外部变量的读取和修改的权限控制。这个是其中一个不同点。

本文链接： [https://feilong.tech/solidity-keyword](https://feilong.tech/solidity-keyword)