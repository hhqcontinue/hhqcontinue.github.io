---
layout: post
title:  "PHP面试题总结"
date:   '2015-04-01'
tags: tech
author: hoohack
categories: PHP
excerpt: 'PHP面试,PHP,面试总结'
keywords: 'PHP面试,PHP,面试总结'
---

去年校园招聘的时候自己准备了挺久的，其中在PHP开发这个岗位上做的准备工作比较多，今天整理自己的印象笔记，发现当时收集了很多资料，在这里整理一下，帮助自己回顾一些基础知识，同时也分享给有需要的人。

##Q:用PHP打印出前一天的时间,格式是2014-5-10 19:20:21

A:

    echo date('Y-m-d H:i:s', strtotime("-1 day"));//method 1
    echo date('Y-m-d H:i:s', time() - 60*60*24);//method 2



##Q:echo(),print,print_r()的区别

A:

>echo是语言结构，无返回值;

>print的功能和echo基本相同，不同的是print是函数，有返回值

>print_r是递归打印，用于输出数组或对象

##Q:用PHP写出显示客户端IP与服务器IP的代码

A:

    $ip=gethostbyname ("");
    echo $ip;

    //获取真实IP
    function getIp() {
        static $realip;
        if (isset($_SERVER)) {
            if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
                $realip = $_SERVER['HTTP_X_FORWARDED_FOR'];
            } else if (isset($_SERVER['HTTP_CLIENT_IP'])) {
                $realip = $_SERVER['HTTP_CLIENT_IP'];
            } else {
                $realip = $_SERVER['REMOTE_ADDR'];
            }
        } else {
            if (getenv('HTTP_X_FORWARDED_FOR')) {
                $realip = getenv('HTTP_X_FORWARDED_FOR');
            } else if (getenv('HTTP_CLIENT_IP')) {
                $realip = getenv('HTTP_CLIENT_IP');
            } else {
                $realip = getenv('REMOTE_ADDR');
            }
        }
        return $realip;
    }
    echo getIP();

##Q:foo() 与 @foo() 有什么分别？

A:

>foo() 会执行这个函式，任何解译错误、语法错误、执行错误都会在页面上显示出来。
@foo() 在执行这个函式时，会隐藏所有上述的错误讯息。
很多应用程式都使用 @mysql_connect() 和 @mysql_query 来隐藏 mysql 的错误讯息，我这其实是很严重的失误，因为错误不该被隐藏，必须妥善处理它们，可能的话解决它们。

##Q:"==="是什么？试举一个"=="是真但"==="是假的例子。

A:

>“===”是给既可以送回布尔值“假”，也可以送回一个不是布尔值但却可以赋与“假”值的函式，strpos() 和 strrpos() 便是其中两个例子。
例子：
if (strpos(“abc”, “a”) == true){ // 这部分永不会被执行，因为 “a” 的位置是 0，换算成布尔值“假”}if (strpos(“abc”, “a”) === true){ // 这部份会被执行，因为“===”保证函式 strpos() 的送回值不会换算成布尔值.}

##Q:哪一个函数可以把浏览器转向到另一个页面？

A:

>header() 可以用来使浏览器转向到另一个页面，例如：
`header(“Location: https://www.hoohack.me/”);`

##Q:include 和 include_once 、require 和 require_once有什么分别？

A:

[`PHP中require、include、require_once和include_once的区别`](https://www.hoohack.me/2015/01/10/php-require-include-require_once-include_once/)

##Q:mysql_fetch_row() 和 mysql_fetch_array() 有什么分别？

A:

>mysql_fetch_row()把数据库的一行数据存储在一个以零为基数的阵列中，第一栏在阵列的索引 0，第二栏在索引 1，如此类推。
mysql_fetch_assoc() 把数据库的一行数据储存在一个关联阵列中，阵列的索引就是栏位名称，例如我的数据库查询送回“first_name”、“last_name”、“email”三个栏位，阵列的索引便是“first_name”、“last_name”和“email”。
mysql_fetch_array() 可以同时送回 mysql_fetch_row() 和 mysql_fetch_assoc() 的值。

##Q:PHP中哪一个函数可以用来开启档案以便读/写

A:

>fopen()这是正确答案，fopen() 可以用来开启档案以便读/写

##Q:sort()、asort()、arsort和 ksort() 有什么分别？它们分别在什么情况下使用？

A:

sort() 函数用于对数组单元从低到高进行排序，如果成功则返回 TRUE，失败则返回 FALSE。

>注意：本函数会为排序的数组中的单元赋予新的键名，这将删除原有的键名而不仅是重新排序。

asort() 函数用于对数组单元从低到高进行排序并保持索引关系，如果成功则返回 TRUE，失败则返回 FALSE。

ksort() 函数用于对数组单元按照键名从低到高进行排序，如果成功则返回 TRUE，失败则返回 FALSE。
本函数会保留原来的键名，因此常用于关联数组。

arsort() 函数行为与 asort() 相反，对数组单元进行由高到低排序并保持索引关系

##Q:写出一个正则表达式，过虑网页上的所有JS脚本（即把script标记及其内容都去掉）

A:

    echo preg_replace("/].*?>.*?/si", "", $script);

##Q:php://input和$_POST有什么区别

A:

首先当`$_POST`与`php://input`可以取到值时$HTTP_RAW_POST_DATA 为空;
$HTTP_RAW_POST_DATA是PHP内置的一个全局变量。它用于，PHP在无法识别的Content-Type的情况下，将POST过来的数据原样地填入变量$HTTP_RAW_POST_DATA。它同样无法读取Content-Type为multipart/form-data的POST数据。需要设置php.ini中的always_populate_raw_post_data值为On，PHP才会总把POST数据填入变量$HTTP_RAW_POST_DATA。

然后$_POST以关联数组方式组织提交的数据，并对此进行编码处理，如urldecode，甚至编码转换;
而php://input 通过输入流以文件读取方式取得未经处理的POST原始数据;

php://input 允许读取 POST 的原始数据。和 $HTTP_RAW_POST_DATA 比起来，它给内存带来的压力较小，并且不需要任何特殊的 php.ini 设置。php://input 不能用于enctype="multipart/form-data";的情况。

php://input读取不到`$_GET`数组的数据。是因为`$_GET`数据作为query_path写在http请求头部(header)的PATH字段，而不是写在http请求的body部分。

##Q:禁用COOKIE 后 SEESION 还能用吗?

A:

session是在服务器端保持用户会话数据的一种方法，对应的cookie是在客户端保持用户数据。HTTP协议是一种无状态协议，服务器响应完之后就失去了与浏览器的联系，最早，Netscape将cookie引入浏览器，使得数据可以客户端跨页面交换，那么服务器是如何记住众多用户的会话数据呢？
首先要将客户端和服务器端建立一一联系，每个客户端都得有一个唯一标识，这样服务器才能识别出来。建议唯一标识的方法有两种：cookie或者通过GET方式指定。默认配置的PHP使用session的时会建立一个名叫”PHPSESSID”的cookie（可以通过php.ini修改session.name值指定），如果客户端禁用cookie,你也可以指定通过GET方式把session id传到服务器（修改php.ini中session.use_trans_sid等参数）。`<a href=”p.php?<?php print session_name() ?>=<?php print session_id() ?>”>xxx</a>`,也可以通过POST来传递session值.

##Q:error_reporting(2047)什么作用

A:

相当于 error_reporting('E_ALL'); 输出所有的错误。

##Q:请写一个函数，实现以下功能：

>字符串"open_door"转换成 "OpenDoor"、"make_by_id" 转换成"MakeById"。

A:

    function str_change($str) {
        $str = str_replace ( "_", " ", $str );
        $str = ucwords ( $str );
        $str = str_replace ( " ", "", $str );
        return $str;
    }

##Q:不用新变量直接交换现有两个变量的值

A:

    list($a,$b)=array($b,$a);//1
    a=a+b,b=a-b,a=a-b;//2
    
##Q:一个函数的参数不能是对变量的引用，除非在php.ini中把?设为on 

    allow_call_time_pass_reference boolean //是否启用在函数调用时强制参数被按照引用传递
    
##Q:PHP的垃圾收集机制是怎样的
>PHP作为脚本语言是页面结束即释放变量所占内存的。 当一个 PHP线程结束时，当前占用的所有内存空间都会被销毁，当前程序中所有对象同时被销毁。GC进程一般都跟着每起一个SESSION而开始运行的.
gc目的是为了在session文件过期以后自动销毁删除这些文件. 在PHP中，没有任何变量指向这个对象时，这个对象就成为垃圾。PHP会将其在内存中销毁；这是PHP 的GC垃圾处理机制，防止内存溢出。 
执行这些函数也可以起到回收作用 `__destruct` /unset/mysql_close /fclose. php对session有明确的gc处理时间设定 session.gc_maxlifetime 如果说有垃圾，那就是整体的程序在框架使用中，
会多次调用同一文件等等造成的非单件模式等。所以在出来的时候，必要的用_once 引用，在声明类的时候使用单件模式。还有简化逻辑等等。而如果妄想让PHP自己本身管理内存，进行垃圾管理。
呵呵。好像PHP还办不到，对于析构函数，ANDI在他的书里写的很明白。可有可无，不可置否。而内存管理的东西一般都是桌面程序更多去考虑的。



To be continue...
