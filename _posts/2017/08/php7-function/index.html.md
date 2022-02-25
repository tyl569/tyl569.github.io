---
layout: post
title: PHP7 拓展编写--在拓展中调用函数
date: 2017-08-24
tags: ["PHP"]
---

PHP中调用函数都比较简单，今天要实现如下效果的代码

    <?php
    class demo {
        public function get_site_name($prefix)
        {
            return $prefix . "肥龙的博客\n";
        }
    }

    function get_site_url($prefix)
    {
        return $prefix . "www.feilong.tech\n";
    }

    function call_function($obj, $fun, $param)
    {
        if ($obj == null)
        {
            $result = $fun($param);
        }
        else
        {
            $result = $obj->$fun($param);
        }
        return $result;
    }
    $demo = new demo();
    echo call_function($demo, "get_site_name", "site name:");
    echo call_function(null, "get_site_url", "site url:");
    ?>

<!--more-->

我们将要实现call_function的方法的功能

#### 代码实现

    PHP_FUNCTION(call_function)
    {
        zval            *obj = NULL;
        zval             *fun = NULL;
        zval             *param = NULL;
        zval             retval;
        zval             args[1];

    #ifndef FAST_ZPP
        /* Get function parameters and do error-checking. */
        if (zend_parse_parameters(ZEND_NUM_ARGS(), "zzz", &obj, &fun, ¶m) == FAILURE) {
            return;
        }
    #else
        ZEND_PARSE_PARAMETERS_START(3, 3)
            Z_PARAM_ZVAL(obj)
            Z_PARAM_ZVAL(fun)
            Z_PARAM_ZVAL(param)
        ZEND_PARSE_PARAMETERS_END();
    #endif

        args[0] = *param;
        if (obj == NULL '' Z_TYPE_P(obj) == IS_NULL) {
            call_user_function_ex(EG(function_table), NULL, fun, &retval, 1, args, 0, NULL);
        } else {
            call_user_function_ex(EG(function_table), obj, fun, &retval, 1, args, 0, NULL);
        }
        RETURN_ZVAL(&retval, 0, 1);
    }

#### 代码解读

<table>
<thead>
<tr>
<th style="text-align: center;">`zend_parse_parameters`
在PHP7中提供了两种获取参数的方法。`zend_parse_parameters`和`FAST ZPP`方式.
在PHP7之前一直使用zend_parse_parameters函数获取参数。这个函数的作用，就是把传入的参数转换为PHP内核中相应的类型，方便在PHP扩展中使用。
`zend_parse_parameters(ZEND_NUM_ARGS(), type_spec, &param1, &param2)`
`ZEND_NUM_ARGS()`代表参数的个数，
`type_spec`代表参数的类型：具体的类型如下</th>
<th style="text-align: center;">参数</th>
<th>对应数据类型</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center;">'</td>
<td style="text-align: center;">之后的参数是可选。可以传，也可以不传</td>
</tr>
<tr>
<td style="text-align: center;">b</td>
<td style="text-align: center;">Boolean</td>
</tr>
<tr>
<td style="text-align: center;">l</td>
<td style="text-align: center;">long</td>
</tr>
<tr>
<td style="text-align: center;">d</td>
<td style="text-align: center;">double</td>
</tr>
<tr>
<td style="text-align: center;">s</td>
<td style="text-align: center;">String 字符串</td>
</tr>
<tr>
<td style="text-align: center;">r</td>
<td style="text-align: center;">Resource 资源</td>
</tr>
<tr>
<td style="text-align: center;">a</td>
<td style="text-align: center;">Array 数组</td>
</tr>
<tr>
<td style="text-align: center;">o</td>
<td style="text-align: center;">Object instance 对象</td>
</tr>
<tr>
<td style="text-align: center;">O</td>
<td style="text-align: center;">Object instance of a specified type 特定类型的对象</td>
</tr>
<tr>
<td style="text-align: center;">z</td>
<td style="text-align: center;">Non-specific zval 任意类型</td>
</tr>
<tr>
<td style="text-align: center;">Z</td>
<td style="text-align: center;">zval**类型</td>
</tr>
</tbody>
</table>

`fast zpp`
在PHP7中新提供的方式。是为了提高参数解析的性能。对应经常使用的方法，建议使用    `FAST ZPP` 方式。 使用方式： 以 `ZEND_PARSE_PARAMETERS_START(1, 2)` 开头。 第一个参数表示必传的参数个数，第二个参数表示最多传入的参数个数。 `以ZEND_PARSE_PARAMETERS_END();`结束。 中间是传入参数的解析。

值得注意的是，一般FAST ZPP的宏方法与  `zend_parse_parameters` 的specifier是一一对应的。如：
`Z_PARAM_OPTIONAL` 对应` '`
`Z_PARAM_STR` 对应` S`
`Z_PARAM_ZVAL` 对应 `z`
但是，`Z_PARAM_ZVAL_EX`方法比较特殊。它对应两个specifier，分别是 ! 和 / 。! 对应宏方法的第二个参数。/ 对应宏方法的第三个参数。如果想开启，只要设置为1即可。

`call_user_function_ex`

`call_user_function_ex`方法用于调用函数和方法。参数说明如下：

第一个参数：方法表。通常情况下，写 `EG(function_table) `
第二个参数：对象。如果不是调用对象的方法，而是调用函数，填写NULL
第三个参数：方法名。
第四个参数：返回值。
第五个参数：参数个数。
第六个参数：参数值。是一个zval数组。
第七个参数：参数是否进行分离操作。
第八个参数：符号表。一般情况写设置为NULL即可。