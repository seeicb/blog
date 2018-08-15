title: DVWA练习2-CSRF
date: 2016-01-03 01:44:17
categories: 安全
tags: Web安全

---
### 一、CSRF
#### 1.CSRF简介
CSRF跨站请求伪造，是一种使已登录用户在不知情的情况下执行某种动作的攻击。因为攻击者看不到伪造请求的响应结果，所以CSRF攻击主要用来执行动作，而非窃取用户数据。当受害者是一个普通用户时，CSRF可以实现在其不知情的情况下转移用户资金、发送邮件等操作；但是如果受害者是一个具有管理员权限的用户时CSRF则可能威胁到整个Web系统的安全。
<!--more-->
#### 2.CSRF类型
CSRF主要有GET型和POST型，还有Flash CSRF。 GET型与POST型CSRF主要取决于相应操作对提交方式的限制，其原理都是事先构造出一个恶意的请求，
然后诱导用户点击或访问，从而假借用户身份完成相应的操作。另外，有些POST型CSRF也可能会利用javascript进行自动提交表单完成操作。Flash CSRF通常是由于Crossdomain.xml文件配置不当造成的，利用方法是使用swf来发起跨站请求伪造。
#### 3.CSRF防御
1.验证码  
2.Referer Check。但是，这种方法存在一些问题需要考虑：首先，Referer的值是由浏览器提供的，虽然HTTP协议上有明确的要求，但是每个浏览器对于
Referer的具体实现可能有差别，并且不能保证浏览器自身没有安全漏洞，将安全性交给第三方（即浏览器）保证，从理论上来讲是不可靠的；其次，用户可能会出于保护隐私等原因禁止浏览器提供Referer，这样的话正常的用户请求也可能因没有Referer信息被误判为不不安全的请求，无法提供正常的使用。   
3.Anti CSRF Token

### 二、DVWA练习
#### 1.low
源码：
```php
<?php
if( isset( $_GET[ 'Change' ] ) ) {
    // Get input
    $pass_new  = $_GET[ 'password_new' ];
    $pass_conf = $_GET[ 'password_conf' ];
    // Do the passwords match?
    if( $pass_new == $pass_conf ) {
        // They do!
        $pass_new = mysql_real_escape_string( $pass_new );
        $pass_new = md5( $pass_new );
        // Update the database
        $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
        $result = mysql_query( $insert ) or die( '<pre>' . mysql_error() . '</pre>' );
        // Feedback for the user
        echo "<pre>Password Changed.</pre>";
    }
    else {
        // Issue with passwords matching
        echo "<pre>Passwords did not match.</pre>";
    }
    mysql_close();
}
?>
```
没有任何防护，直接构造链接，诱使用户点击即可。
```html
<img src="http://127.0.0.1/dvwa/vulnerabilities/csrf/?password_new=passwordlow&password_conf=passwordlow&Change=Change#"/>
```
即可更改密码。
#### 2.Medium
源码：
```php
<?php
if( isset( $_GET[ 'Change' ] ) ) {
    // Checks to see where the request came from
    if( eregi( $_SERVER[ 'SERVER_NAME' ], $_SERVER[ 'HTTP_REFERER' ] ) ) {
        // Get input
        $pass_new  = $_GET[ 'password_new' ];
        $pass_conf = $_GET[ 'password_conf' ];
        // Do the passwords match?
        if( $pass_new == $pass_conf ) {
            // They do!
            $pass_new = mysql_real_escape_string( $pass_new );
            $pass_new = md5( $pass_new );
            // Update the database
            $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
            $result = mysql_query( $insert ) or die( '<pre>' . mysql_error() . '</pre>' );
            // Feedback for the user
            echo "<pre>Password Changed.</pre>";
        }
        else {
            // Issue with passwords matching
            echo "<pre>Passwords did not match.</pre>";
        }
    }
    else {
        // Didn't come from a trusted source
        echo "<pre>That request didn't look correct.</pre>";
    }
    mysql_close();
}
?>
```
`if( eregi( $_SERVER[ 'SERVER_NAME' ], $_SERVER[ 'HTTP_REFERER' ] ) )`
这里使用验证HTTP Referer字段进行防御。
~~关于Referer的绕过网上给出了一些方法，http://www.cnblogs.com/huangjacky/p/4109096.html~~
~~1.通过地址栏，手动输入；从书签里面选择；通过实现设定好的手势。上面说的这三种都是用户自己去操作，因此不算CSRF。2.跨协议间提交请求。常见的协议：ftp:,http:,https:,file:,javascript:,data:.最简单的情况就是我们在本地打开一个HTML页面，这个时候浏览器地址栏是file://开头的，如果这个HTML页面向任何http站点提交请求的话，这些请求的Referer都是空的。那么我们接下来可以利用data:协议来构造一个自动提交的CSRF攻击。当然这个协议是IE不支持的，我们可以换用javascript。~~

更新：这里判断Referer是使用eregi函数进行判断。只需要在Referer中包含有$_SERVER[ 'SERVER_NAME' ]就可以通过。所i可以新建一个html页面为 http://xxx.com/127.0.0.1.html ，包含有修改密码链接，诱使用户点击即可。

```
<img src="http://127.0.0.1/dvwa/vulnerabilities/csrf/?password_new=passwordlow&password_conf=passwordlow&Change=Change#"/>
```



#### 3.High
代码：
```php
<?php
if( isset( $_GET[ 'Change' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // Get input
    $pass_new  = $_GET[ 'password_new' ];
    $pass_conf = $_GET[ 'password_conf' ];
    // Do the passwords match?
    if( $pass_new == $pass_conf ) {
        // They do!
        $pass_new = mysql_real_escape_string( $pass_new );
        $pass_new = md5( $pass_new );
        // Update the database
        $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";
        $result = mysql_query( $insert ) or die( '<pre>' . mysql_error() . '</pre>' );
        // Feedback for the user
        echo "<pre>Password Changed.</pre>";
    }
    else {
        // Issue with passwords matching
        echo "<pre>Passwords did not match.</pre>";
    }
    mysql_close();
}
// Generate Anti-CSRF token
generateSessionToken();
?>
```
`checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );`
这里检查Token来验证。
更新：这里陷入了思维误区，一直想着在这个页面上突破，但是token在这里是得不到的。需要同存储型XSS联合起来，得到token的值。然后构造链接。

参考：[DVWA-1.9全级别教程之CSRF](http://www.freebuf.com/articles/web/118352.html)
PS：在记录完学习DVWA过程后，看到的文章，解决了我的几个问题，看完之后只感觉我好蠢。



#### 4.Impossible
```php
<?php
if( isset( $_GET[ 'Change' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // Get input
    $pass_curr = $_GET[ 'password_current' ];
    $pass_new  = $_GET[ 'password_new' ];
    $pass_conf = $_GET[ 'password_conf' ];
    // Sanitise current password input
    $pass_curr = stripslashes( $pass_curr );
    $pass_curr = mysql_real_escape_string( $pass_curr );
    $pass_curr = md5( $pass_curr );
    // Check that the current password is correct
    $data = $db->prepare( 'SELECT password FROM users WHERE user = (:user) AND password = (:password) LIMIT 1;' );
    $data->bindParam( ':user', dvwaCurrentUser(), PDO::PARAM_STR );
    $data->bindParam( ':password', $pass_curr, PDO::PARAM_STR );
    $data->execute();
    // Do both new passwords match and does the current password match the user?
    if( ( $pass_new == $pass_conf ) && ( $data->rowCount() == 1 ) ) {
        // It does!
        $pass_new = stripslashes( $pass_new );
        $pass_new = mysql_real_escape_string( $pass_new );
        $pass_new = md5( $pass_new );

        // Update database with new password
        $data = $db->prepare( 'UPDATE users SET password = (:password) WHERE user = (:user);' );
        $data->bindParam( ':password', $pass_new, PDO::PARAM_STR );
        $data->bindParam( ':user', dvwaCurrentUser(), PDO::PARAM_STR );
        $data->execute();
        // Feedback for the user
        echo "<pre>Password Changed.</pre>";
    }
    else {
        // Issue with passwords matching
        echo "<pre>Passwords did not match or current password incorrect.</pre>";
    }
}

// Generate Anti-CSRF token
generateSessionToken();
?>
```
Token检查和二次验证。
