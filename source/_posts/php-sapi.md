---
title: 'PHP 底层: SAPI概述'
tags: []
id: '133'
categories:
  - - Linux
date: 2017-08-24 20:08:44
---

### 概述

> 各个服务器抽象层遵守着相同的规定，统一称为SAPI接口。而SAPI接口的格式由一个\_sapi\_module\_struct的结构体定义好。在PHP中，如果需要调用服务器的信息，统一通过SPAI接口进行实现。

<!--more-->

下面是SAPI调用的简单示意图

![](/uploads/2017/07/02-02-01-sapi.png)
<!-- more -->
以CGI模式和apache2为例，启动方式如下：

```cpp
.... // 上面都是初始化启动前的赋值操作
/* startup after we get the above ini override se we get things right */
cgi_sapi_module.startup(&cgi_sapi_module)   //  cgi模式 cgi/cgi_main.c文件 main方法内

apache2_sapi_module.startup(&apache2_sapi_module);
 //  apache2服务器  apache2handler/sapi_apache2.c文件 php_apache_server_startup方法

// 虽然方法不一样，但是使用的都是一个相同的结构体 _sapi_module_struct
```

这里的cgi\_sapi\_module和apache2\_sapi\_module都是\_sapi\_module\_struct格式的静态变量。cgi\_sapi\_module的startup方法指向php\_cgi\_startup函数指针。

![](/uploads/2017/08/14994211711.png)

### \_sapi\_module\_struct结构体

在结构体\_sapi\_module\_struct中除了startup函数指针，还有许多其它方法或字段。

结构体大概是如下的格式：

```cpp
struct _sapi_module_struct {
    char *name;
    char *pretty_name;

    int (*startup)(struct _sapi_module_struct *sapi_module);
    int (*shutdown)(struct _sapi_module_struct *sapi_module);

    int (*activate)(TSRMLS_D);
    int (*deactivate)(TSRMLS_D);

    int (*ub_write)(const char *str, unsigned int str_length TSRMLS_DC);
    void (*flush)(void *server_context);
    struct stat *(*get_stat)(TSRMLS_D);
    char *(*getenv)(char *name, size_t name_len TSRMLS_DC);

    void (*sapi_error)(int type, const char *error_msg, ...);

    int (*header_handler)(sapi_header_struct *sapi_header, sapi_header_op_enum op, sapi_headers_struct *sapi_headers TSRMLS_DC);
    int (*send_headers)(sapi_headers_struct *sapi_headers TSRMLS_DC);
    void (*send_header)(sapi_header_struct *sapi_header, void *server_context TSRMLS_DC);

    int (*read_post)(char *buffer, uint count_bytes TSRMLS_DC);
    char *(*read_cookies)(TSRMLS_D);

    void (*register_server_variables)(zval *track_vars_array TSRMLS_DC);
    void (*log_message)(char *message);
    time_t (*get_request_time)(TSRMLS_D);
    void (*terminate_process)(TSRMLS_D);

    char *php_ini_path_override;

    void (*block_interruptions)(void);
    void (*unblock_interruptions)(void);

    void (*default_post_reader)(TSRMLS_D);
    void (*treat_data)(int arg, char *str, zval *destArray TSRMLS_DC);
    char *executable_location;

    int php_ini_ignore;

    int (*get_fd)(int *fd TSRMLS_DC);

    int (*force_http_10)(TSRMLS_D);

    int (*get_target_uid)(uid_t * TSRMLS_DC);
    int (*get_target_gid)(gid_t * TSRMLS_DC);

    unsigned int (*input_filter)(int arg, char *var, char **val, unsigned int val_len, unsigned int *new_val_len TSRMLS_DC);

    void (*ini_defaults)(HashTable *configuration_hash);
    int phpinfo_as_text;

    char *ini_entries;
    const zend_function_entry *additional_functions;
    unsigned int (*input_filter_init)(TSRMLS_D);
};
```

其中一些函数指针的说明如下：

*   **startup 当SAPI初始化时，首先会调用该函数。如果服务器处理多个请求时，该函数只会调用一次。 比如Apache的SAPI，它是以mod\_php5的Apache模块的形式加载到Apache中的， 在这个SAPI中，startup函数只在父进程中创建一次，在其fork的子进程中不会调用。**
*   **activate 此函数会在每个请求开始时调用，它会再次初始化每个请求前的数据结构。**
*   **deactivate 此函数会在每个请求结束时调用，它用来确保所有的数据都，以及释放在activate中初始化的数据结构。**
*   **shutdown 关闭函数，它用来释放所有的SAPI的数据结构、内存等。**
*   **ub\_write 不缓存的写操作(unbuffered write)，它是用来将PHP的数据输出给客户端， 如在CLI模式下，其最终是调用fwrite实现向标准输出输出内容；在Apache模块中，它最终是调用Apache提供的方法rwrite。**
*   **sapi\_error 报告错误用，大多数的SAPI都是使用的PHP的默认实现php\_error。**
*   **flush 刷新输出，在CLI模式下通过使用C语言的库函数fflush实现，在php\_mode5模式下，使用Apache的提供的函数函数rflush实现。**
*   **read\_cookie 在SAPI激活时，程序会调用此函数，并且将此函数获取的值赋值给SG(request\_info).cookie\_data。 在CLI模式下，此函数会返回NULL。**
*   **read\_post 此函数和read\_cookie一样也是在SAPI激活时调用，它与请求的方法相关，当请求的方法是POST时，程序会操作\\$\_POST、\\$HTTP\_RAW\_POST\_DATA等变量。**
*   **send\_header 发送头部信息，此方法一般的SAPI都会定制，其所不同的是，有些的会调服务器自带的（如Apache），有些的需要你自己实现（如 FastCGI）。**

以上的这些结构在各服务器的接口实现中都有定义。如Apache2的定义：

```cpp
static sapi_module_struct apache2_sapi_module = {
    "apache2filter",                       /* name */
    "Apache 2.0 Filter",                   /* pretty_name*/

    php_apache2_startup,                        /* startup */
    php_module_shutdown_wrapper,            /* shutdown */

    NULL,                                   /* activate */
    NULL,                                   /* deactivate */

    php_apache_sapi_ub_write,               /* unbuffered write */
    php_apache_sapi_flush,                  /* flush */
    php_apache_sapi_get_stat,                       /* get uid */
    php_apache_sapi_getenv,                 /* getenv */

    php_error,                              /* error handler */

    php_apache_sapi_header_handler,         /* header handler */
    php_apache_sapi_send_headers,           /* send headers handler */
    NULL,                                   /* send header handler */

    php_apache_sapi_read_post,              /* read POST data */
    php_apache_sapi_read_cookies,           /* read Cookies */

    php_apache_sapi_register_variables,
    php_apache_sapi_log_message,            /* Log message */
    php_apache_sapi_get_request_time,       /* Get Request Time */
    NULL,                       /* Child terminate */

    STANDARD_SAPI_MODULE_PROPERTIES
};
```

整个SAPI类似于一个面向对象中的模板方法模式的应用。 SAPI.c和SAPI.h文件所包含的一些函数就是模板方法模式中的抽象模板， 各个服务器对于sapi\_module的定义及相关实现则是一个个具体的模板。

这样的结构在PHP的源码中有多处使用， 比如在PHP扩展开发中，每个扩展都需要定义一个zend\_module\_entry结构体。 这个结构体的作用与sapi\_module\_struct结构体类似，都是一个类似模板方法模式的应用。 在PHP的生命周期中如果需要调用某个扩展，其调用的方法都是zend\_module\_entry结构体中指定的方法， 如在上一小节中提到的在执行各个扩展的请求初始化时，都是统一调用request\_startup\_func方法， 而在每个扩展的定义时，都通过宏PHP\_RINIT指定request\_startup\_func对应的函数。 以VLD扩展为例：其请求初始化为PHP\_RINIT(vld),与之对应在扩展中需要有这个函数的实现：

```cpp
PHP_RINIT_FUNCTION(vld) {
}
```

所以， 我们在写扩展时也需要实现扩展的这些接口，同样，当实现各服务器接口时也需要实现其对应的SAPI。