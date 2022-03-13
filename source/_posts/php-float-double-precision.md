---
title: PHP float & double 序列化导致的精度问题
tags:
  - json
  - PHP
  - PHP7
  - PHP源码
  - PHP精度
id: '484'
categories:
  - - PHP
  - - PHP源码
date: 2019-09-06 15:38:40
---

### 问题发现

Json\_encode应该算是PHP函数中，使用次数最多的函数之一了，尤其是在一些api接口定义，我们通常会使用一些json进行返回。但是在一次开发中，我发现在返回浮点型数字的时候，数字出奇的长。在使用var\_dump的时候，明明是正常的，怎么接口返回的时候就变异了？？？？

<!--more-->

```php
// 实例代码

$list = [
 '100','200','5','100.23',200.222222,'5','100','200','1'
];
echo json_encode($list);

[100,200,5,100.23,200.2222219999999879291863180696964263916015625,5,100,200,1]
```

我们可以看到，经过`json_encode`之后，浮点型的小数，边长了，出现了精度的问题。

### 调试源码

PHP的所有的内置的函数，都是通过拓展的形式安装和运行的。json\_encode也不例外，拓展就是 ext/json

```c
// ext/json/json.c

static PHP_FUNCTION(json_encode)
{
    zval *parameter;
    smart_str buf = {0};
    zend_long options = 0;
    zend_long depth = PHP_JSON_PARSER_DEFAULT_DEPTH;

    if (zend_parse_parameters(ZEND_NUM_ARGS(), zll, &parameter, &options, &depth) == FAILURE) {
        return;
    }

    JSON_G(error_code) = PHP_JSON_ERROR_NONE;

    JSON_G(encode_max_depth) = (int)depth;

    php_json_encode(&buf, parameter, (int)options);

    if (JSON_G(error_code) != PHP_JSON_ERROR_NONE && !(options & PHP_JSON_PARTIAL_OUTPUT_ON_ERROR)) {
        smart_str_free(&buf);
        ZVAL_FALSE(return_value);
    } else {
        smart_str_0(&buf); /* copy? */
        ZVAL_NEW_STR(return_value, buf.s);
    }
}
....
PHP_JSON_API void php_json_encode(smart_str *buf, zval *val, int options) /* {{{ */
{
    php_json_encode_zval(buf, val, options);
}
```

PHP\_FUNCTION 里面是对`json_encode`的函数的定义，首先就是解析 `json_encode` 的参数列表，和选项设置，然后，调用了`php_json_encode`的方法，执行json编码的主要操作，并把一些 [json options](https://www.php.net/manual/zh/json.constants.php) 作为参数传进去。 所以，我们的重点，就是调试 `php_json_encode`的执行过程。

### 运行调试

```shell
$ lldb php7
(lldb) target create php7
Current executable set to 'php7' (x86_64).
(lldb) b php_json_encode
Breakpoint 1: where = php7`php_json_encode + 19 at json.c:196:23, address = 0x000000010027e963
(lldb) r debug/json.php
Process 31145 launched: '/usr/local/bin/php7' (x86_64)
Process 31145 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x000000010027e963 php7`php_json_encode(buf=0x00007ffeefbfe2a0, val=0x0000000101a16110, options=0) at json.c:196:23
   193 
   194  PHP_JSON_API void php_json_encode(smart_str *buf, zval *val, int options) /* {{{ */
   195  {
- 196 php_json_encode_zval(buf, val, options); 
    197 } 
    198 /* }}} */ 
    199 
```

我们可以打印看看传输的值，是不是和我们填写的参数一致

```shell
(lldb) frame variable val->value.arr->nNumOfElements (uint32_t) val->value.arr->nNumOfElements = 9 
```

我们输出了数组的元素的个数，正好是9个元素。

### 步步紧逼

```c
// json_encoder.c 
int php_json_encode_zval(smart_str *buf, zval *val, int options, php_json_encoder *encoder) /* {{{ */
{
again:
    switch (Z_TYPE_P(val))
    {
        case IS_NULL:
            smart_str_appendl(buf, "null", 4);
            break;

        case IS_TRUE:
            smart_str_appendl(buf, "true", 4);
            break;
        case IS_FALSE:
            smart_str_appendl(buf, "false", 5);
            break;

        case IS_LONG:
            smart_str_append_long(buf, Z_LVAL_P(val));
            break;

        case IS_DOUBLE:
            if (php_json_is_valid_double(Z_DVAL_P(val))) {
                php_json_encode_double(buf, Z_DVAL_P(val), options);
            } else {
                encoder->error_code = PHP_JSON_ERROR_INF_OR_NAN;
                smart_str_appendc(buf, '0');
            }
            break;

        case IS_STRING:
            return php_json_escape_string(buf, Z_STRVAL_P(val), Z_STRLEN_P(val), options, encoder);

        case IS_OBJECT:
            if (instanceof_function(Z_OBJCE_P(val), php_json_serializable_ce)) {
                return php_json_encode_serializable_object(buf, val, options, encoder);
            }
            /* fallthrough -- Non-serializable object */
        case IS_ARRAY: // 如果是数组类型 
            return php_json_encode_array(buf, val, options, encoder);

        case IS_REFERENCE:
            val = Z_REFVAL_P(val);
            goto again;

        default:
            encoder->error_code = PHP_JSON_ERROR_UNSUPPORTED_TYPE;
            if (options & PHP_JSON_PARTIAL_OUTPUT_ON_ERROR) {
                smart_str_appendl(buf, "null", 4);
            }
            return FAILURE;
    }

    return SUCCESS;
}
/* }}} */
```

由于我们的参数类型是数组，所以，在判断val的类型的时候，就会跳到 case IS\_ARRAY，然后开始执行 php\_json\_encode\_array

```c
// json_encoder.c 

static int php_json_encode_array(smart_str *buf, zval *val, int options, php_json_encoder *encoder) /* {{{ */
{
    int i, r, need_comma = 0;
    HashTable *myht;

    if (Z_TYPE_P(val) == IS_ARRAY) {
        myht = Z_ARRVAL_P(val);
        r = (options & PHP_JSON_FORCE_OBJECT) ? PHP_JSON_OUTPUT_OBJECT : php_json_determine_array_type(val);
    } else { // 判断是否设置强制转换成json对象，即json_encode的option是否为JSON_FORCE_OBJECT 
        myht = Z_OBJPROP_P(val);
        r = PHP_JSON_OUTPUT_OBJECT;
    }
    if (myht && ZEND_HASH_GET_APPLY_COUNT(myht) > 1) {
        encoder->error_code = PHP_JSON_ERROR_RECURSION;
        smart_str_appendl(buf, "null", 4);
        return FAILURE;
    }

    if (r == PHP_JSON_OUTPUT_ARRAY) {
        smart_str_appendc(buf, '[');  // 如果是json数组，那么就是[开头
    } else {
        smart_str_appendc(buf, '{'); // 如果是json对象，那么就是{开头
    }
    ++encoder->depth;
    i = myht ? zend_hash_num_elements(myht) : 0; // 统计数组中元素的个数：i=9 
    ... 代码省略
    if (i > 0) {
        zend_string *key;
        zval *data;
        zend_ulong index;
        HashTable *tmp_ht;

        ZEND_HASH_FOREACH_KEY_VAL_IND(myht, index, key, data) { // 对数组的元素进行解析 
        ... 代码省略
        }  ZEND_HASH_FOREACH_END();

    }
    ... 代码省略

    if (r == PHP_JSON_OUTPUT_ARRAY) {
        smart_str_appendc(buf, ']'); // 追加json字符串的结束符
    } else {
        smart_str_appendc(buf, '}'); // 追加json字符串的结束符
    }
}
```

在对数组的每个元素进行编码的时候会重复的执行`php_json_encode_zval`的witch的判断。 当循环到 200.222222，那么就是走到`case IS_DOUBLE:`的分支，然后执行`php_json_encode_double`的方法

```c
static inline void php_json_encode_double(smart_str *buf, double d, int options) /* {{{ */
{
    size_t len;
    char num[PHP_DOUBLE_MAX_LENGTH];

    php_gcvt(d, (int)PG(serialize_precision), '.', 'e', num); // 根据数字和配置的precision长度，截取数字，赋值给num
    len = strlen(num);
    if (options & PHP_JSON_PRESERVE_ZERO_FRACTION && strchr(num, '.') == NULL && len < PHP_DOUBLE_MAX_LENGTH - 2) {
        num[len++] = '.';
        num[len++] = '0';
        num[len] = '\0';
    }
    smart_str_appendl(buf, num, len); // 将截取后的num追加到json字符串后面
}
```

### 总结

json-encode的执行还算是比较容易理解

1.  先获取待编码的zval
2.  判断zval的类型，针对不同的类型，进行不同的编码
3.  如果是数组，那么就先按个遍历数组的元素，递归进行编码
4.  当元素是浮点型的时候，根据配置文件里面的 serialize\_precision或precision进行截取
5.  截取后的数字追加到json字符串的尾部

根据php.ini文档的说明

> ; The number of significant digits displayed in floating point numbers.
> 
> ; When floats & doubles are serialized store serialize\_precision significant ; digits after the floating point. The default value ensures that when floats ; are decoded with unserialize, the data will remain the same. ; The value is also used for json\_encode when encoding double values. ; If -1 is used, then dtoa mode 0 is used which automatically select the best ; precision.

设置serialize\_precision=-1 将会自动使用最适合的长度，即我们经常使用的浮点型长度。

所以，我们最好设置-1来设置序列化长度

```php
ini_set('precision', -1);
ini_set('serialize_precision', -1);
```

文章链接: [http://feilong.tech/2019/09/06/php-float-double-precision](http://feilong.tech/2019/09/06/php-float-double-precision)