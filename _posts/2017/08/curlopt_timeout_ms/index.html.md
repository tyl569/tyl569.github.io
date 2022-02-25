---
layout: post
title: CURL 设置CURLOPT_TIMEOUT_MS 超时时间
date: 2017-08-24
tags: ["Linux"]
---

#### 作为PHPer，基本上经常会使用curl函数，调用接口，爬取数据，有时候接口返回时间太长的话，会影响业务逻辑。

#### 比如：我调用天气接口，如果天气接口处理时间很长，比如5s，这就严重影响到业务了。所以针对这种情况，我们需要直接忽略。

##### 最近在调用图灵接口，最长返回时间高达4s，我们的期望是时间超过500毫秒，就直接忽略。

_curl可以通过设置变量 `CURLOPT_TIMEOUT_MS`或者 `CURLOPT_TIMEOUT`设置超时时间。_

> CURLOPT_TIMEOUT_MS 以毫秒计算超时时间
> CURLOPT_TIMEOUT 以秒计算超时时间

查了手册关于`timeout`的设置

<table>
<thead>
<tr>
<th style="text-align: center;">变量</th>
<th style="text-align: center;">说明</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center;">CURLOPT_TIMEOUT</td>
<td style="text-align: center;">允许 cURL 函数执行的最长秒数。</td>
</tr>
<tr>
<td style="text-align: center;">CURLOPT_TIMEOUT_MS</td>
<td style="text-align: center;">设置cURL允许执行的最长毫秒数。 如果 libcurl 编译时使用系统标准的名称解析器（ standard system name resolver），那部分的连接仍旧使用以秒计的超时解决方案，最小超时时间还是一秒钟。</td>
</tr>
</tbody>
</table>

##### 假设我们想要设置超时时间是500毫秒，那么直接设置 curl_setopt($ch, CURLOPT_TIMEOUT_MS, 500);

但是，测试的时候，发现，设置了500ms，curl直接返回了false，并且打印了下错误信息：`cURL Error (28): Timeout was reached` 可能和libcurl 的编译有关系，我继续查了下php手册，发现了如下的内容

> If you want cURL to timeout in less than one second, you can use CURLOPT_TIMEOUT_MS, although there is a bug/"feature"  on "Unix-like systems" that causes libcurl to timeout immediately if the value is < 1000 ms with the error "cURL Error (28): Timeout was reached".  The explanation for this behavior is:
> "If libcurl is built to use the standard system name resolver, that portion of the transfer will still use full-second resolution for timeouts with a minimum timeout allowed of one second."
> 
> What this means to PHP developers is "You can use this function without testing it first, because you can't tell if libcurl is using the standard system name resolver (but you can be pretty sure it is)"
> 
> The problem is that on (Li'U)nix, when libcurl uses the standard name resolver, a SIGALRM is raised during name resolution which libcurl thinks is the timeout alarm.
> 
> The solution is to disable signals using CURLOPT_NOSIGNAL.  Here's an example script that requests itself causing a 10-second delay so you can test timeouts:

内容的大概意思是：libcurl使用的是标准名称的解析器，内部传输信息号部分，依然是使用秒来计算是否超时，并且最少时间是1秒，如果想要设置超时时间小于1秒，那么直接通过`CURLOPT_NOSIGNAL`禁用信号即可。即：`curl_setopt($ch, CURLOPT_NOSIGNAL, 1);`

    #demo code

     if (!isset($_GET['foo'])) {
            // Client
          $ch = curl_init('http://localhost/test/test_timeout.php?foo=bar');
          curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
          curl_setopt($ch, CURLOPT_NOSIGNAL, 1);
          curl_setopt($ch, CURLOPT_TIMEOUT_MS, 200);
          $data = curl_exec($ch);
          $curl_errno = curl_errno($ch);
          $curl_error = curl_error($ch);
          curl_close($ch);

            if ($curl_errno > 0) {
                    echo "cURL Error ($curl_errno): $curl_error\n";
            } else {
                   echo "Data received: $data\n";
            }
    } else {
            // Server
            sleep(10);
            echo "Done.";
    }

#### 本文连接：[http://feilong.tech/2017/08/24/curlopt_timeout_ms](http://feilong.tech/2017/08/24/curlopt_timeout_ms)