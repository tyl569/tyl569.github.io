---
layout: post
title: 15个PHP开发者常用的正则表达式及实例
date: 2017-08-24
tags: ["PHP"]
---

> 从字符串中删除特定字符
> 本段代码实现删除字符串中除所有大小写字母及数字以外的字符

    <?php
    $value = "wWw.UncleToo.Com - 【UncleToo中文网】 - 12345";
    $value = preg_replace("/[^A-Za-z0-9]/","",$value);
    echo $value;
    //输出：wWwUncleTooComUncleToo12345
    ?>

<!--more-->
> 验证用户名
> 以下代码验证用户名是否由字母、数字及下划线组成。

    <?php
    $username = "uncletoo_COM123";
    if (preg_match('/^[a-z\d_]{5,20}$/i', $username)) {
        echo "用户名可用";
    } else {
        echo "用户名存在特殊字符";
    }
    ?>

> 添加信息到图片alt属性
> 使用下面函数，可以实现将文章标题添加到图片的alt属性中。

    <?php
    function add_alt_tags($content) {
        global $post;
        preg_match_all('//', $content, $images);
        if(!is_null($images)) {
            foreach($images[1] as $index => $value) {
                if(!preg_match('/alt=/', $value)) {
                    $new_img = str_replace('

> 将EMail文本自动添加Mailto链接

    <?php
    $text = "demo@abc.com";
    $string = eregi_replace('([_\.0-9a-z-]+@([0-9a-z][0-9a-z-]+\.)+[a-z]{2,3})','<a href="mailto:\\1">\\1</ a>', $text);
    echo $string;
    ?>

> 过滤限制级词语

    <?php
    function filtrado($texto, $reemplazo = false) {
        $filtradas = 'admin,uncletoo,中文网'; //这里定义需要过滤的词语
        $f = explode(',', $filtradas);
        $f = array_map('trim', $f);
        $filtro = implode(''', $f);
        return ($reemplazo) ? preg_replace("#$filtro#i", $reemplazo, $texto) : preg_match("#$filtro#i", $texto) ;
    }
    ?>

> 验证电话号码
> 这是一个很常见的功能

    <?php
    $string = "(010) 555-5555";
    if (preg_match('/^\(?[0-9]{3}\)?'[0-9]{3}[-. ]? [0-9]{3}[-. ]?[0-9]{4}$/', $string)) {
       echo "successful.";
    }
    ?>

> 替换超链接href属性的内容
> 在页面上查看源文件，显示为：< a href="yes" >UncleToo中文网< /a>

    <?php
    $html = '<a href="http://www.uncletoo.com">UncleToo中文网</a>';
    $replacement = "yes";
    $pattern = '/(?<=href\=")[^]]+?(?=")/';
    $replacedHrefHtml = preg_replace($pattern, $replacement, $html);
    echo $replacedHrefHtml ;
    ?>

> 验证邮箱正则表达式
> 此功能在用户注册是经常使用

    <?php
    $regex = "([a-z0-9_.-]+)". # name
    "@". # at
    "([a-z0-9.-]+){2,255}". # domain & possibly subdomains
    ".". # period
    "([a-z]+){2,10}"; # domain extension
    $eregi = eregi_replace($regex, '', $email);
    $valid_email = empty($eregi) ? true : false;
    ?>

> IP地址验证

    <?php
    $string = "255.255.255.255";
    if (preg_match(
    '/^(?:25[0-5]'2[0-4]\d'1\d\d'[1-9]\d'\d)(?:[.](?:25[0-5]'2[0-4]\d'1\d\d'[1-9]\d'\d)){3}$/', $string)) {
        echo "IP address is good.";
    }
    ?>

> 邮政编码验证

    <?php
    $string = "12345-1234";
    if (preg_match('/^[0-9]{5}([- ]?[0-9]{4})?$/', $string)) {
        echo "zip code checks out";
    }
    ?>

> 高亮显示文本

    <?php
    $text = "UncleToo（www.uncletoo.com）中文网";
    $text = preg_replace("/\b(www)\b/i", '<span style="background:#5fc9f6">\1</ span>',$text);
    echo $text;
    ?>

> 从特定的URL中提取域名

    <?php
    $url = "http://www.uncletoo.com/plug/tags/?tag=PHP";
    preg_match('@^(?:http://)?([^/]+)@i', $url, $matches);
    $host = $matches[1];
    echo $host;
    //输出：www.uncletoo.com
    ?>

> 验证域名格式是否正确

    <?php
    $url = "http://www.uncletoo.com/";
    if (preg_match('/^(http'https'ftp):\/\/([A-Z0-9][A-Z0-9_-]*(?:\.[A-Z0-9][A-Z0-9_-]*)+):?(\d+)?\/?/i', $url)) {
        echo "域名格式正确.";
    } else {
        echo "域名格式错误.";
    }
    ?>

> 使用文章标题生成URL

    <?php
    function create_slug($string){
       $slug=preg_replace('/[^A-Za-z0-9-]+/', '-', $string);
       return $slug;
    }
    echo create_slug('my name is uncletoo');
    //输出：my-name-is-uncletoo
    ?>

> 添加http://到URL地址
> 当我们需要用户填写网址时，很多用户往往不填写http://直接输入域名，使用下面代码可将http://添加到网址的前面。

    <?php
    if (!preg_match("/^(http'https'ftp):/", $_POST['url'])) {
       $_POST['url'] = 'http://'.$_POST['url'];
    }
    ?>

> 将URL转换为超链接
> 这时一个很有用的功能，他可以将url地址或email地址转换为可点击的超链接文本。

    <?php
    function makeLinks($text) {
        $text = eregi_replace('(((f'ht){1}tp://)[-a-zA-Z0-9@:%_+.~#?&//=]+)','\1', $text);
        $text = eregi_replace('([[:space:]()[{}])(www.[-a-zA-Z0-9@:%_+.~#?&//=]+)','\1\2',$text);
        $text = eregi_replace('([_.0-9a-z-]+@([0-9a-z][0-9a-z-]+.)+[a-z]{2,3})','\1', $text);
        return $text;
    }
    ?>