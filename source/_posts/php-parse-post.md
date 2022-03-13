---
title: PHP $_POST接收大量form表单数据缺失探究
tags:
  - http
  - PHP
  - PHP7
id: '730'
categories:
  - - PHP
  - - PHP源码
date: 2019-12-20 21:39:55
---

### 背景

最近遇到一个线上问题，服务A，调用服务B的接口，发现服务B报“xxx参数不存在”，但是通过服务A的请求日志发现，是有参数"xxx"。然后翻了一下服务B的日志，发现没有参数"xxx"，而且以外发现，接收的数据，比传输的数据少一部分！

<!--more-->

### 黑人问号？？

最初怀疑是A传输写数据的原因，随后在请求前，打印了内容，`发现是完整的！！！好玄幻！！！`

然后在B打印了 file\_get\_contents("php://input")，$\_POST，发前者的内容是完整的，后者的内容要偏少，所以A传输的数据应该没有问题！！

那问题到底出现在了哪里？？？

### 探究原因

为了查明下原因，我怀疑是和代码有关系，所以索性，把数据搞到postman，通过postman再尝试下

```php
#接收端代码
<?php
file_put_contents("txt", json_encode($_POST));
```

```bash
# 启动phpserver服务
$ php7 -S 127.0.0.1:9090
PHP 7.1.2RC1 Development Server started at Fri Dec 20 13:40:31 2019
Listening on http://127.0.0.1:9090
Document root is /Users/sf
Press Ctrl-C to quit.
[Fri Dec 20 13:40:38 2019] PHP Warning:  Unknown: Input variables exceeded 1000. To increase the limit change max_input_vars in php.ini. in Unknown on line 0
[Fri Dec 20 13:40:38 2019] 127.0.0.1:59719 [200]: /index.php
```

意外发现，控制台输出了一条warning信息

`[Fri Dec 20 13:40:38 2019] PHP Warning: Unknown: Input variables exceeded 1000. To increase the limit change max_input_vars in php.ini. in Unknown on line 0`

这条信息，仿佛是一棵救命稻草一样，我根据提示信息，查询了一下官方文档

> max\_input\_vars integer 接受多少 输入的变量（限制分别应用于 $\_GET、$\_POST 和 $\_COOKIE 超全局变量） 指令的使用减轻了以哈希碰撞来进行拒绝服务攻击的可能性。 如有超过指令指定数量的输入变量，将会导致 E\_WARNING 的产生， 更多的输入变量将会从请求中截断。

原来，PHP处于安全考虑，会在form表单提交的时候，会限制参数解析的个数，如果超过规定的个数，就会出现截断的问题，默认限制是1000个。而我提交的数据，早就超过1000个了。这应该就是$\_POST要比php://input里面数据少的原因了。

### 截取的策略

原因找到了，这就萌生了另外一个问题，$\_POST虽然被截断了，但是为什么打印出来的信息，还是一个完整的数组的结构？截断的策略是什么样的？这恐怕需要分析一下PHP的源码了！

```bash
#php_variables.c

static inline int add_post_vars(zval *arr, post_var_data_t *vars, zend_bool eof)
{
    uint64_t max_vars = PG(max_input_vars);

    vars->ptr = ZSTR_VAL(vars->str.s);
    vars->end = ZSTR_VAL(vars->str.s) + ZSTR_LEN(vars->str.s);
    while (add_post_var(arr, vars, eof)) {
        if (++vars->cnt > max_vars) {
            php_error_docref(NULL, E_WARNING,
                    "Input variables exceeded %" PRIu64 ". "
                    "To increase the limit change max_input_vars in php.ini.",
                    max_vars);
            return FAILURE;
        }
    }

    if (!eof) {
        memmove(ZSTR_VAL(vars->str.s), vars->ptr, ZSTR_LEN(vars->str.s) = vars->end - vars->ptr);
    }
    return SUCCESS;
}
```

PHP会首先初始化解析数据的指针，然后通过while的循环，逐次对post的数据进行解析，然后设置vars->ptr的值，用来记录当前解析的位置，并对解析变量的个数进行统计，当`++vars->cnt > max_vars` 的时候，会终止解析，但是解析的时候，是按照key value结对解析，所以截断后，$\_POST里面也是标准的数组结构。

`add_post_var`是具体的解析策略， 以`name=feilong&sex=man`为例

```bash
static zend_bool add_post_var(zval *arr, post_var_data_t *var, zend_bool eof)
{
    char *ksep, *vsep, *val;
    size_t klen, vlen;
    size_t new_vlen;

    if (var->ptr >= var->end) {
        return 0;
    }

    vsep = memchr(var->ptr, '&', var->end - var->ptr);
    if (!vsep) {
        if (!eof) {
            return 0;
        } else {
            vsep = var->end;
        }
    }

    ksep = memchr(var->ptr, '=', vsep - var->ptr);
    if (ksep) {
        *ksep = '\0';
        /* "foo=bar&" or "foo=&" */
        klen = ksep - var->ptr;
        vlen = vsep - ++ksep;
    } else {
        ksep = "";
        /* "foo&" */
        klen = vsep - var->ptr;
        vlen = 0;
    }

    php_url_decode(var->ptr, klen);

    val = estrndup(ksep, vlen);
    if (vlen) {
        vlen = php_url_decode(val, vlen);
    }

    if (sapi_module.input_filter(PARSE_POST, var->ptr, &val, vlen, &new_vlen)) {
        php_register_variable_safe(var->ptr, val, new_vlen, arr);
    }
    efree(val);

    var->ptr = vsep + (vsep != var->end);
    return 1;
}
```

首先，定位到`name=feilong`，然后进行拆解，将name赋值var->ptr，将feilong赋值给val变量，然后通过`php_register_variable_safe`函数，进行变量的注册。

不过在注册之前，会先进行一次过滤的操作。

> filter\_input filter\_input — 通过名称获取特定的外部变量，并且可以通过过滤器处理它 filter\_input ( int $type , string $variable\_name \[, int $filter = FILTER\_DEFAULT \[, mixed $options \]\] ) : mixed type INPUT\_GET, INPUT\_POST, INPUT\_COOKIE, INPUT\_SERVER或 INPUT\_ENV之一。 .....

从源码上来看，这里过滤的类型是`INPUT_POST`。

PHP就是通过这样一层层的循环解析，直到解析的变量个数超过限制或者解析结束，把原有的form表单的数据，解析成$\_POST数组。

### 总结

回到文章的标题，其实这次问题出现的原因，并不是在于"传输"，而在于解析数据，处于安全考虑，PHP做了一层限制，防止黑客传输过多的数据，导致用户被DDoS攻击。

本文链接: [https://feilong.tech/2019/12/20/php-parse-post](https://feilong.tech/2019/12/20/php-parse-post)