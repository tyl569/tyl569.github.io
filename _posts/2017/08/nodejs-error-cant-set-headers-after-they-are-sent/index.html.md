---
layout: post
title: NodeJs Error: Can''t set headers after they are sent.怎么解决？
date: 2017-08-24
tags: ["Nodejs"]
---

从字面的意思来说：不能发送header，因为已经发送过一次了。

我的程序之所以出现这种情况，是因为多次使用`res.sent()`

原因：在处理HTTP请求时，服务器会先输出响应头，然后再输出主体内容，而一旦输出过一次响应头（比如执行过 `res.writeHead()` 或 `res.write()` 或 `res.end()`），你再尝试通过 `res.setHeader()` 或 `res.writeHead()` 来设置响应头时（有些方法比如 `res.redirect()` 会调用 `res.writeHead()`），就会报这个错误。

解决办法：在调用函数后面加上`return;` 终止，这样就搞定了。