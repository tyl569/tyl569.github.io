---
title: PHP curl传送json数据
tags: []
id: '136'
categories:
  - - PHP
date: 2017-08-24 20:11:54
---

### PHP curl传送json数据

#### 说句实话，写PHP也有几年了，不过感觉技术还是渣的不行。前段时间给公司的机器人写接口，客户端的哥哥打算用json上报数据。这可憋闷死我了，因为以前都是直接使用form表单的形式，也就是键值对。没办法，只能双方商量一下，改用键值对的方式了。（大写的尴尬）

#### 今天终于找到了使用json互传数据的办法了。用到的主要函数是curl，file\_get\_contents()和json\_decode()。
<!-- more -->
#### 1)定义数据格式

```php
$params = array(
    'name' => 'Tyler Teng',
    'sex'  => 'male'
);
$params = json_encode($params);
```

#### 2)创建curl句柄，并且采用post方式进行传输

```php
$url    =  'http://localhost/get.php';
$ch = curl_init();
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_POST, 1);
curl_setopt($ch, CURLOPT_POSTFIELDS, $params);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 0);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0);
$out = curl_exec($ch);
curl_close($ch);
```

#### 3) 一般情况下，我们会在get.php使用

```php
print_r($_POST);
//string 'Array()' (length=10)
```

这个时候一定是空的，因为现在的数据不是以键值对传输的，而是使用数据流进行传输。所以应该使用

```php
print_r(file_get_contents('php://input'));
//string '{"name":"Tyler Teng","sex":"male"}' (length=34)
```

来检验一下是不是有数据。

#### 4)使用json\_decode()函数进行解析。

```php
$json = file_get_contents('php://input');
$array = json_decode($json, true);
```

#### 5)解析后的json就是数组了，我们只需要按照数组方式进行获取数值就好了

```php
echo $array['name'];
//string 'Tyler Teng' (length=10)
```