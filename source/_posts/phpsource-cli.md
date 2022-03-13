---
title: PHP源码分析之cli模式执行的过程
tags:
  - PHP源码
id: '180'
categories:
  - - PHP
  - - PHP源码
comments: false
date: 2017-09-26 20:50:15
---

众所周知，PHP在web上应用很广泛。接近80%的web网站都是使用PHP+MySQL，虽然越来越多的新语种崛起，但是现在PHP依然是中小型web系统的首选。PHP除了在web上有很多应用，也经常被用作脚本工具，虽然没有原生shell效率高，但是起点比较低。今天就和大家分享下PHP cli模式的执行过程。

<!-- more -->

#### 前期准备

```
PHP的版本：5.3
代码查看工具：VSCODE 或者sublime text
```

#### 一切从SAPI开始

`SAPI`是`Server Application Programming Interface`的缩写，翻译过来就是服务应用程序接口，可以理解称是一种接口的规范，只要是符合规范的语言，都是可以通过SAPI和服务器进行数据交互。

通常，在web模式下，PHP通常都是运行在Apache或者nginx这类web服务器上面，程序执行结束后，将结果显示在浏览器上面。其实命令行和web执行过程是稍微有点不同的。命令行是将参数传给PHP的解释器，然后把运行结果显示在窗口上面。有兴趣的可以阅读下[深入PHP内核：用户代码的执行](http://www.php-internals.com/book/?p=chapt02/02-01-php-life-cycle-and-zend-engine) 了解一下PHP的生命周期。

PHP的cli模式最开始是随着PHP 4.2.0的版本发布的，但是当时只是一个实验版本，并且需要使用`./configure --enable-cli`参数才会进行安装。直到PHP 4.3.0之后，才把cli模式当成正式的模块，`--enable-cli` 参数会被默认得设置为 on，也可以用参数 `--disable-cli` 来屏蔽。

#### 入口的位置

当我们忘记一个命令的option的时候，我们通常会使用`-h/--help`来查看帮助

```bash
    [root@root ~]$ php -h
    Usage: php [options] [-f] <file> [--] [args...]
       php [options] -r <code> [--] [args...]
       php [options] [-B <begin_code>] -R <code> [-E <end_code>] [--] [args...]
       php [options] [-B <begin_code>] -F <file> [-E <end_code>] [--] [args...]
       php [options] -- [args...]
       php [options] -a

      -a               Run as interactive shell
      -c <path><file> Look for php.ini file in this directory
      -n               No php.ini file will be used
      -d foo[=bar]     Define INI entry foo with value 'bar'
      -e               Generate extended information for debugger/profiler
      -f <file>        Parse and execute <file>.
      -h               This help
      -i               PHP information
      -l               Syntax check only (lint)
      -m               Show compiled in modules
      -r <code>        Run PHP <code> without using script tags <?..?>
      -B <begin_code>  Run PHP <begin_code> before processing input lines
      -R <code>        Run PHP <code> for every input line
      -F <file>        Parse and execute <file> for every input line
      -E <end_code>    Run PHP <end_code> after processing all input lines
      -H               Hide any passed arguments from external tools.
      -s               Output HTML syntax highlighted source.
      -v               Version number
      -w               Output source with stripped comments and whitespace.
      -z <file>        Load Zend extension <file>.

      args...          Arguments passed to script. Use -- args when first argument
                       starts with - or script is read from stdin

      --ini            Show configuration file names

      --rf <name>      Show information about function <name>.
      --rc <name>      Show information about class <name>.
      --re <name>      Show information about extension <name>.
      --ri <name>      Show configuration for extension <name>.
```

以上就是PHP的命令已经一些参数。 在`/sapi/cli/php_cli.c`文件里面有个`main`方法，可以说这个方法就是程序的入口位置了。

#### 运行的流程

从代码可以看得出来，这个过程大概可以分为：

*   参数的处理
*   cli\_sapi\_module的初始化
*   cli\_sapi\_module的启动(starup)
*   函数的执行
*   垃圾回收
*   输出信息

##### 参数的处理

PHP的命令可以接受一系列的参数，比如常见的`php -i`或者`php -m`等等，传递给全局变量`$argv` ，该数组中下标为零的成员为脚本的名称（当 PHP 代码来自标准输入获直接用 -r 参数以命令行方式运行时，该名称为"-"）。另外，全局变量 \\$argc 存有 \\$argv 数组中成员变量的个数（而非传送给脚本程序的参数的个数）。

对于参数的解析，可以查看下PHP的源码 `/sapi/cli/php_cli.c` 大概725行左右

```cpp
    .....
    while ((c = php_getopt(argc, argv, OPTIONS, &php_optarg, &php_optind, 0, 2))!=-1) { //对参数进行解析
        switch (c) {
            case 'c':
                if (cli_sapi_module.php_ini_path_override) {
                    free(cli_sapi_module.php_ini_path_override);
                }
    .....
```

完整的解析方法就是`php_getopt`，在`/main/getopt.c` 的第58行左右，在php\_getopt方法里面，通过对 '-' 或者 '--' 的处理，获取具体的参数，然后返回。

```cpp
PHPAPI int php_getopt(int argc, char* const *argv, const opt_struct opts[], char **optarg, int *optind, int show_err, int arg_start) 
```

##### cli\_sapi\_module的初始化

其实cli\_sapi\_module的初始化和参数的处理两个过程的先后并不是很明显，因为在参数处理之前，也有一些简单的初始化操作，比如对cli模式下的PHP配置文件的初始化，因为在使用cli命令的时候是需要一些初始化的值才行。

```cpp
    cli_sapi_module.ini_defaults = sapi_cli_ini_defaults;
    cli_sapi_module.php_ini_path_override = NULL;
    cli_sapi_module.phpinfo_as_text = 1;
    sapi_startup(&cli_sapi_module);
```

我之所以放到后面，是因为大部分的成员变量初始化都是在参数处理之后的。

cli\_sapi\_module是一个静态全局变量，数据结构比较容易理解

```cpp
static sapi_module_struct cli_sapi_module = {
    "cli",                            /* name */
    "Command Line Interface",     /* pretty name */

    php_cli_startup,                /* startup */
    php_module_shutdown_wrapper,    /* shutdown */

    NULL,                           /* activate */
    sapi_cli_deactivate,            /* deactivate */

    sapi_cli_ub_write,              /* unbuffered write */
    sapi_cli_flush,                 /* flush */
    NULL,                           /* get uid */
    NULL,                           /* getenv */
    .....
```

**其实伴随着cli\_sapi\_module初始化，PHP也会对模块进行启动的操作**

```cpp
static int php_cli_startup(sapi_module_struct *sapi_module) /* {{{ */
{
    if (php_module_startup(sapi_module, NULL, 0)==FAILURE) {
        return FAILURE;
    }
    return SUCCESS;
}
```

##### cli\_sapi\_module启动(startup)

启动的过程比较简单明白，如果启动失败的话，那就goto错误信息处理阶段，在控制台输出错误信息

```cpp
    /* startup after we get the above ini override se we get things right */
    if (cli_sapi_module.startup(&cli_sapi_module)==FAILURE) {
        /* there is no way to see if we must call zend_ini_deactivate()
         * since we cannot check if EG(ini_directives) has been initialised
         * because the executor's constructor does not set initialize it.
         * Apart from that there seems no need for zend_ini_deactivate() yet.
         * So we goto out_err.*/
        exit_status = 1;
        goto out_err;
    }
```

##### 函数的执行

启动结束后，PHP会根据参数不同，调用不同的函数，比如当用户输入`php -i`的时候，那么就打印出PHP的info信息；输入`php -m`的时候打印出已经安装的模块...

```cpp
while ((c = php_getopt(argc, argv, OPTIONS, &php_optarg, &php_optind, 0, 2)) != -1) {
    switch (c) {
        ......              
        case 'i': /* php info & quit */
            if (php_request_startup(TSRMLS_C)==FAILURE) { ## 请求初始化操作
                goto err;
            }
            request_started = 1;
            php_print_info(0xFFFFFFFF TSRMLS_CC);
            php_end_ob_buffers(1 TSRMLS_CC);
            exit_status=0;
            goto out;
        case 'm': /* list compiled in modules */
            if (php_request_startup(TSRMLS_C)==FAILURE) {
                goto err;
            }
            request_started = 1;
            php_printf("[PHP Modules]\n");
            print_modules(TSRMLS_C);
            php_printf("\n[Zend Modules]\n");
            print_extensions(TSRMLS_C);
            php_printf("\n");
            php_end_ob_buffers(1 TSRMLS_CC);
            exit_status=0;
            goto out;
        case 'v': /* show php version & quit */
            if (php_request_startup(TSRMLS_C) == FAILURE) {
                goto err;
            }
            request_started = 1;
            php_printf("PHP %s (%s) (built: %s %s) %s\nCopyright (c) 1997-2010 The PHP Group\n%s",
            PHP_VERSION, sapi_module.name, __DATE__, __TIME__,
            ....
```

此外，根据对参数的switch的case的比较，确定behavior （解释器行为）根据解释器行为，然后根据不同的behavior 做出想用的动作。

```cpp
...
case PHP_MODE_LINT:
    exit_status = php_lint_script(&file_handle TSRMLS_CC);
    if (exit_status==SUCCESS) {
        zend_printf("No syntax errors detected in %s\n", file_handle.filename);
    } else {
        zend_printf("Errors parsing %s\n", file_handle.filename);
    }
    break;
case PHP_MODE_STRIP:
    if (open_file_for_scanning(&file_handle TSRMLS_CC)==SUCCESS) {
        zend_strip(TSRMLS_C);
    }
    goto out;
    break;
....
```

伴随着不同的解释器行为，进行请求的处理

```cpp
    if (php_request_startup(TSRMLS_C)==FAILURE) {
        *arg_excp = arg_free;
        fclose(file_handle.handle.fp);
        PUTS("Could not startup.\n");
        goto err;
    }
```

##### 垃圾回收

在代码的执行过程中，PHP会通过全局函数`CG()`或者函数`free()`将内存和数据进行释放，进行垃圾的回收。

##### 输出信息

运行的最后应该就是对信息的输出和对SAPI的关闭。这部分其实和web请求类似，输出（错误信息）之后，PHP会通过`php_request_shutdown`，`php_module_shutdown`和`sapi_shutdown`等对相应的请求、模块和SAPI等进行关闭。**但是和web请求不一样的是，每次结束cli模式的时候都是会对模块进行关闭(`php_module_shutdown`)，但是web模式缺不是，web模式在PHP启动和关闭的时候才会知心模块的初始化以及关闭，并是不每处理完一个请求就开启/关闭一次。**

#### 总结

cli模式和web模式其实大同小异，整个PHP的生命周期基本一致：开始->模块初始化->请求初始化->执行PHP脚本->关闭请求->关闭模块。但是最大的不同是因为是否重复的启动，因为在web模式下，往往是连续的请求，也就是通常用户经常做页面的跳转，如果重复的启动也关闭模块，势必会造成性能上的差异。但是cli模式往往都是单次的请求，是不连续的。

#### 文献参考

[深入理解PHP的内核](http://www.php-internals.com)

[PHP的命令行模式](http://www.php100.com/manual/php/features.commandline.html)