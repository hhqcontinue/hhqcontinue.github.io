---
layout: post
title:  "MySQL中VARCHAR与CHAR格式数据的区别"
date:   '2015-01-04'
tags: blog
author: hoohack
categories: MySQL
---

#区别
CHAR与VARCHAR类型类似，但它们保存和检索的方式不同。CHAR有固定的长度，而VARCHAR属于可变长的字符类型。它们最大长度和是否尾部空格被保留等方面也不同。在存储和检索过程中不进行大小写转换。

下面的表格显示了将各种字符串值保存到CHAR(4)和VARCHAR(4)列后的结果，说明了CHAR和VARCHAR之间的差别：
<table>
    <thead>
        <th>值&nbsp;&nbsp;&nbsp;</th>
        <th>CHAR(4)</th>
        <th>存储需求</th>
        <th>VARCHAR(4)</th>
        <th>存储需求</th>
    </thead>
    <tbody>
        <tr>
            <td>''</td>
            <td>'    '</td>
            <td>4个字节</td>
            <td>''</td>
            <td>1个字节</td>
        </tr>
        <tr>
            <td>'ab'</td>
            <td>'ab  '</td>
            <td>4个字节</td>
            <td>'ab'</td>
            <td>3个字节</td>
        </tr>
        <tr>
            <td>'abcd'</td>
            <td>'abcd'</td>
            <td>4个字节</td>
            <td>'abcd'</td>
            <td>5个字节</td>
        </tr>
        <tr>
            <td>'abcdefgh'</td>
            <td>'abcd'</td>
            <td>4个字节</td>
            <td>'abcd'</td>
            <td>5个字节</td>
        </tr>
    </tbody>
</table>

从上面可以看得出来CHAR的长度是固定的，不管你存储的数据是多少他都会都固定的长度。而VARCHAR则处可变长度但他要在总长度上加1字符，这个用来存储位置。所以实际应用中用户可以根据自己的数据类型来做。

请注意，上表中最后一行的值只适用不使用严格模式时;如果MySQL运行在严格模式，超过列长度的值不被保存，并且会出现错误。

从CHAR(4)和VARCHAR(4)列检索的值并不总是相同，因为检索时从CHAR列删除了尾部的空格。通过下面的例子说明差别：

    mysql> CREATE TABLE test(a VARCHAR(4), b CHAR(4));

    mysql> INSERT INTO test VALUES ('ab  ', 'ab  ');

    mysql> SELECT CONCAT(a, '+'), CONCAT(b, '+') FROM test;

结果如下：
<table>
    <thead>
        <th>CONCAT(a, '+')</th>
        <th>CONCAT(b, '+')</th>
    </thead>
    <tbody>
        <tr>
            <td>ab  +</td>
            <td>ab+</td>
        </tr>
    </tbody>
</table>

从上面可以看出来，由于某种原因CHAR有固定长度，所以在处理速度上要比VARCHAR快很多，但是相对浪费存储空间，所以对存储不大，但在速度上有要求的可以使用CHAR类型，反之可以用VARCHAR类型来实现。
 
#建议

> * MyISAM存储引擎 建议使用固定长度，数据列代替可变长度的数据列 
> * INNODB 存储引擎 建议使用VARCHAR类型 
