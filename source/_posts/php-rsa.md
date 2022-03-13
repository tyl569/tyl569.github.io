---
title: PHP实现RSA加密、解密操作
tags: []
id: '138'
categories:
  - - PHP
date: 2017-08-24 20:16:45
---

#### [RSA生成工具](https://os.alipayobjects.com/download/secret_key_tools_RSA_win.zip)

#### 现在越来越多的人注重安全问题，尤其是在支付过程中，不管是卖家还是买家都希望交易过程中不出现任何差错，顺利进行，没有损失。所以在各大支付接口都在支付过程中加入了签名`sign`验证。

<!-- more -->

#### 签名的作用

#### 签名有个最基本的作用就是安全，你可以把一些数据生成签名一块传过去，接收方通过接收的数据和签名，进行验签，如果验证通过，则继续下一个逻辑。这样做防止传输过程中数据被篡改或者丢失造成的损失。

#### RSA签名是目前最具影响力的公钥加密算法，可以抵制绝大多数密码攻击，而且秘钥比较长，不容易破解。

#### 使用方法

> *   使用私钥进行加密，公钥用于解密
> *   私钥和公钥嘴放放在server上面，并且放在非项目目录，防止泄露

#### 实现代码

```php
//生成签名
function build_sign($data) {
    $private_key = openssl_pkey_get_private('file://C:/key/rsa_private_key.pem');//私钥位置
    openssl_sign($data, $sign, OPENSSL_ALGO_SHA1);
    $sign = base64_encode($sign);

    return $sign;
}
//解密
function check_sign($date, $sign) {
    $sign = base64_decode($sign);
    $public_key = openssl_pkey_get_public('file://C:/key/rsa_public_key.pem');//公钥位置
    $result = openssl_verify($data, $sign, $key, OPENSSL_ALGO_SHA1) == 1;

    return $result;
}
```