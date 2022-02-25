---
layout: post
title: mac 由于libiconv导致编译PHP7 ld: symbol(s) not found for architecture x86_64错误
date: 2019-11-11
tags: ["PHP","PHP","PHP7","PHP源码","PHP源码"]
---

从源码手动编译 PHP 时出现如下错误：

    Undefined symbols for architecture x86_64:
      "_libiconv", referenced from:
          _php_iconv_string in iconv.o
          __php_iconv_strlen in iconv.o
          __php_iconv_substr in iconv.o
          __php_iconv_strpos in iconv.o
          __php_iconv_mime_encode in iconv.o
          __php_iconv_appendl in iconv.o
          _php_iconv_stream_filter_append_bucket in iconv.o
          ...
      "_libiconv_close", referenced from:
          _php_iconv_string in iconv.o
          __php_iconv_strlen in iconv.o
          __php_iconv_substr in iconv.o
          __php_iconv_strpos in iconv.o
          __php_iconv_mime_encode in iconv.o
          __php_iconv_mime_decode in iconv.o
          _php_iconv_stream_filter_dtor in iconv.o
          ...
      "_libiconv_open", referenced from:
          _php_iconv_string in iconv.o
          __php_iconv_strlen in iconv.o
          __php_iconv_substr in iconv.o
          __php_iconv_strpos in iconv.o
          __php_iconv_mime_encode in iconv.o
          __php_iconv_mime_decode in iconv.o
          _php_iconv_stream_filter_ctor in iconv.o
          ...
    ld: symbol(s) not found for architecture x86_64
    clang: error: linker command failed with exit code 1 (use -v to see invocation)
    make: *** [sapi/cli/php] Error 1

这个是因为我在编译的时候设置--with-iconv的路径，猜测应该是iconv的问题。
参照文章 compile php with openssl on mac osx error找到了一些灵感
MakeFile 里面找到类似下面这一行：

    EXTRA_LIBS = -lresolv -liconv -liconv -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm

删除所有 -liconv 在后面填写 libiconv.dylib和libcharset.dylib的路径
如果你是用homebrew安卓的libiconv那么路径就是 /usr/local/opt/libiconv/lib
附上我修改后的 MakeFile EXTRA_LIBS 那一行：

    EXTRA_LIBS = -lresolv -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm -lxml2 -lz -licucore -lm /usr/local/opt/libiconv/lib/libiconv.dylib /usr/local/opt/libiconv/lib/libcharset.dylib

然后重新运行make命令

本文连接：[https://feilong.tech/2019/11/11/make-php-error](https://feilong.tech/2019/11/11/make-php-error)