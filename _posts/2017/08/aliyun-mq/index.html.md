---
layout: post
title: 阿里云消息队列和消息服务的使用
date: 2017-08-24
tags: ["Linux"]
---

### 应用场景

#### 异步处理

消息队列的一的特点之一就是异步处理，这就决定了对于实时返回的信息就没办法使用消息队列。经常使用的消息队列比如发送邮箱验证、短信验证。因为一般的逻辑是串行方式，消息队列采用的是并行的模式。

<!--more-->
> **串行方式**：串行方式基本上是在编程中最常见的方式了，也就是完全按照流程做事。举个例子，我早上起床后，先刷牙（5分钟），然后再吃早饭（20分钟），我使用的时间是20+5分钟

![](%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%971-1.png)

> **并行方式**：并行方式也可以是算是异步处理，这样处理的效率会更快。举个例子，我晚上下班回家，我一边泡脚(5分钟)，泡脚的同时，我还顺便吃了晚饭（20分钟），这个过程我花了20分钟，因为泡脚和吃饭是同时进行的。

![](%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%972-1.png)

消息队列实现方式：

![](%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%974.png)

#### 应用解耦

用户在下单之后，账单系统把账单信息发给库存系统，库存系统进行发货。但是如果库存系统某次访问不了，可能会导致订单失败。

![](%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%975-300x111.png)

消息队列形式：

![](%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%976.png)

#### 流量削峰

很多网站的访问瓶颈大多数都是出现在数据库上面，如果某个接口出现访问量太大，必然会增加压力。或者某个接口调用第三方API，很容易出现数据丢失的状况，这种情景可以使用消息队列。

![](%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%977.png)

### 阿里云消息队列和消息服务

#### 消息服务和消息队列的对比

<table>
<thead>
<tr>
<th style="text-align: center;">对比项目</th>
<th style="text-align: center;">消息服务(MNS,原MQS)</th>
<th style="text-align: center;">消息队列(ONS)</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center;">queue模型</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">官方SDK</td>
<td style="text-align: center;">Java,C++,Python,C#,PHP,Node.js(非官方),Golang(非官方)</td>
<td style="text-align: center;">Java,C/C++,C#,PHP(http),Python(http)</td>
</tr>
<tr>
<td style="text-align: center;">支持JMS</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">No</td>
</tr>
<tr>
<td style="text-align: center;">协议支持</td>
<td style="text-align: center;">HTTP</td>
<td style="text-align: center;">TCP,HTTP,MQTT</td>
</tr>
<tr>
<td style="text-align: center;">延时消息</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">定时消息</td>
<td style="text-align: center;">No</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">事务消息</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">消息Batch操作</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">No</td>
</tr>
<tr>
<td style="text-align: center;">保证消息至少消费一次</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">支持RAM访问控制</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">消息优先级</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">No</td>
</tr>
<tr>
<td style="text-align: center;">消息推拉模式</td>
<td style="text-align: center;">Pull，Push</td>
<td style="text-align: center;">Pull，Push</td>
</tr>
<tr>
<td style="text-align: center;">消息轨迹追踪</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">服务端消息过滤</td>
<td style="text-align: center;">Yes</td>
<td style="text-align: center;">Yes</td>
</tr>
<tr>
<td style="text-align: center;">qps性能</td>
<td style="text-align: center;">默认5000</td>
<td style="text-align: center;">默认5000</td>
</tr>
<tr>
<td style="text-align: center;">数据可靠性</td>
<td style="text-align: center;">99.99999999%</td>
<td style="text-align: center;">99.99%</td>
</tr>
<tr>
<td style="text-align: center;">数据堆积</td>
<td style="text-align: center;">不限</td>
<td style="text-align: center;">不限</td>
</tr>
<tr>
<td style="text-align: center;">服务可用性</td>
<td style="text-align: center;">99.9%</td>
<td style="text-align: center;">99.9%</td>
</tr>
</tbody>
</table>

#### API对比

[消息服务API地址](https://help.aliyun.com/document_detail/27473.html?spm=5176.doc27437.6.226.7LkW7O)

[消息队列http API地址](https://help.aliyun.com/document_detail/29572.html)

> **消息服务的接口相对齐全一些，支持的类型也相对齐全，topic和queue两种形式。不过开发起来比较费劲，因为涉及到[签名](https://help.aliyun.com/document_detail/27487.html?spm=5176.doc27473.6.241.4ffAlt)的过程，在接收的时候，为了保证消息的安全性，需要进行验签，验签通过或才能进行逻辑处理。**
> 
> **消息队列的http接口比较简单，但是消息的重复率较高，达到了20%，官网提供的demo比较简单，只有发送，消费和删除，需要进程循环接收消息，比较消耗资源open API接口齐全，但是暂时没有支持PHP的SDK，综合考虑下选择了消息服务。**