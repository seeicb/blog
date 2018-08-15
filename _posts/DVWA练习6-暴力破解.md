title: DVWA练习6-暴力破解
date: 2016-03-16 13:37:45
categories: 安全  
tags: Web安全  

---

### 一、暴力破解
#### 1.简介
暴力破解是一种针对密码破解的方法，主要是使用枚举法，将密码逐个测试，直到找到真正的密码。一般暴力破解比较耗时。且需要好的字典。
#### 2.攻击
一般的攻击主要是使用爆破软件进行破解，主要的工具有Hydra,burpsuite等等。
<!--more-->
hydra使用方法参见http://www.cnblogs.com/mchina/archive/2013/01/01/2840815.html
#### 3.防御
1.使用验证码，但是验证码也有被破解的可能。
2.尽量使用高强度密码。
3.登陆日志，限制最大错误次数。

### 二、DVWA练习
#### 1. Low
在输入框随意输入用户名和密码，然后抓包,点击send to intruder。
![](http://7xo8y2.com1.z0.glb.clouddn.com/201603170001.png)
![](http://7xo8y2.com1.z0.glb.clouddn.com/201603170002.png)
在intruder页面中，点击positions,配置变量。先点击clear$清楚变量，然后选中用户名和密码，点击add$,把admin和password作为变量,攻击方式为 cluster bomb。
![](http://7xo8y2.com1.z0.glb.clouddn.com/201603170006.png)
点击payloads，配置字典。
payload set对指定变量设置。
payload type指定payload类型，这里选择simple list。
选择payload set 1在payload options中添加列表，这里演示所以添加少量，同样在payload set 2添加密码列表。然后点击start attack。如图
![](http://7xo8y2.com1.z0.glb.clouddn.com/201603170004.png)
攻击结果如下，点击length可以看到不同的返回长度，一般不同的就是正确的用户名和密码。
![](http://7xo8y2.com1.z0.glb.clouddn.com/201603170005.png)
如图，可以看到用户名为admin，密码为password。
关于burpsuite的教程可以参考http://drops.wooyun.org/tools/1548  
查看源码：
```php
<?php
if( isset( $_GET[ 'Login' ] ) ) {
    // Get username用户名
    $user = $_GET[ 'username' ];
    // Get password密码
    $pass = $_GET[ 'password' ];
    $pass = md5( $pass );
    // Check the database 查找
    $query  = "SELECT * FROM `users` WHERE user = '$user' AND password = '$pass';";
    $result = mysql_query( $query ) or die( '<pre>' . mysql_error() . '</pre>' );
    if( $result && mysql_num_rows( $result ) == 1 ) {
        // Get users details
        $avatar = mysql_result( $result, 0, "avatar" );
        // Login successful
        echo "<p>Welcome to the password protected area {$user}</p>";
        echo "<img src=\"{$avatar}\" />";
    }
    else {
        // Login failed
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }
    mysql_close();
}
?>
```
可以看到这里查询用户名时没有过滤，所以也存在SQL注入漏洞  
#### 2. Medium
破解形式同low级别一样，但是使用了sleep函数，登陆失败则延缓执行2秒，所以会增大时间。同时源码中做了SQL注入防御。
```php

<?php

if( isset( $_GET[ 'Login' ] ) ) {
    // Sanitise username input
    $user = $_GET[ 'username' ];
    $user = mysql_real_escape_string( $user );

    // Sanitise password input
    $pass = $_GET[ 'password' ];
    $pass = mysql_real_escape_string( $pass );
    $pass = md5( $pass );

    // Check the database
    $query  = "SELECT * FROM `users` WHERE user = '$user' AND password = '$pass';";
    $result = mysql_query( $query ) or die( '<pre>' . mysql_error() . '</pre>' );

    if( $result && mysql_num_rows( $result ) == 1 ) {
        // Get users details
        $avatar = mysql_result( $result, 0, "avatar" );

        // Login successful
        echo "<p>Welcome to the password protected area {$user}</p>";
        echo "<img src=\"{$avatar}\" />";
    }
    else {
        // Login failed
        sleep( 2 ); //暂停2秒
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }
    mysql_close();
}
?>
```
#### 3. High
~~再次使用burp会发现使用了token，并且每次都是随机的，无法使用爆破。同时使用sleep函数。~~  
更新：虽然无法使用burp，但是可以使用python脚本，自动获取token值，进行爆破。参考[Python模拟登录](http://seeicb.com/2016/05/06/Python模拟登录/)


源码：
```php
<?php

if( isset( $_GET[ 'Login' ] ) ) {
    // Check Anti-CSRF token
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );

    // Sanitise username input
    $user = $_GET[ 'username' ];
    //stripslashes() 函数删除由 addslashes() 函数添加的反斜杠。
    //本函数可去掉字符串中的反斜线字符。若是连续二个反斜线，则去掉一个，留下一个。若只有一个反斜线，就直接去掉。
    $user = stripslashes( $user );
    $user = mysql_real_escape_string( $user );

    // Sanitise password input
    $pass = $_GET[ 'password' ];
    $pass = stripslashes( $pass );
    $pass = mysql_real_escape_string( $pass );
    $pass = md5( $pass );

    // Check database
    $query  = "SELECT * FROM `users` WHERE user = '$user' AND password = '$pass';";
    $result = mysql_query( $query ) or die( '<pre>' . mysql_error() . '</pre>' );

    if( $result && mysql_num_rows( $result ) == 1 ) {
        // Get users details
        $avatar = mysql_result( $result, 0, "avatar" );

        // Login successful
        echo "<p>Welcome to the password protected area {$user}</p>";
        echo "<img src=\"{$avatar}\" />";
    }
    else {
        // Login failed失败随机暂停0到3秒
        sleep( rand( 0, 3 ) );
        echo "<pre><br />Username and/or password incorrect.</pre>";
    }

    mysql_close();
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```

#### 4. Impossible
最大登陆失败次数限制  
源码：
```php
<?php
if( isset( $_POST[ 'Login' ] ) ) {
    // CSRF防御
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    // stripslashes()函数删除由 addslashes() 函数添加的反斜杠。
    $user = $_POST[ 'username' ];
    $user = stripslashes( $user );
    $user = mysql_real_escape_string( $user );
    //防御SQL注入
    // Sanitise password input
    $pass = $_POST[ 'password' ];
    $pass = stripslashes( $pass );
    $pass = mysql_real_escape_string( $pass );
    $pass = md5( $pass );
    // Default values
    $total_failed_login = 3;
    $lockout_time       = 15;
    $account_locked     = false;
    //最大失败次数，失败锁定时间
    // Check the database (Check user information)
    //查询语句
    $data = $db->prepare( 'SELECT failed_login, last_login FROM users WHERE user = (:user) LIMIT 1;' );
    $data->bindParam( ':user', $user, PDO::PARAM_STR );
    $data->execute();
    $row = $data->fetch();
    //查看用户是否被锁定
    // Check to see if the user has been locked out.
    if( ( $data->rowCount() == 1 ) && ( $row[ 'failed_login' ] >= $total_failed_login ) )  {
        // User locked out.  Note, using this method would allow for user enumeration!
        //echo "<pre><br />This account has been locked due to too many incorrect logins.</pre>";

        // Calculate when the user would be allowed to login again
        $last_login = $row[ 'last_login' ];
        $last_login = strtotime( $last_login );
        $timeout    = strtotime( "{$last_login} +{$lockout_time} minutes" );
        $timenow    = strtotime( "now" );

        // Check to see if enough time has passed, if it hasn't locked the account
        if( $timenow > $timeout )
            $account_locked = true;
    }

    // Check the database (if username matches the password)
    $data = $db->prepare( 'SELECT * FROM users WHERE user = (:user) AND password = (:password) LIMIT 1;' );
    $data->bindParam( ':user', $user, PDO::PARAM_STR);
    $data->bindParam( ':password', $pass, PDO::PARAM_STR );
    $data->execute();
    $row = $data->fetch();

    // If its a valid login...
    if( ( $data->rowCount() == 1 ) && ( $account_locked == false ) ) {
        // Get users details
        $avatar       = $row[ 'avatar' ];
        $failed_login = $row[ 'failed_login' ];
        $last_login   = $row[ 'last_login' ];

        // Login successful
        echo "<p>Welcome to the password protected area <em>{$user}</em></p>";
        echo "<img src=\"{$avatar}\" />";

        // Had the account been locked out since last login?
        if( $failed_login >= $total_failed_login ) {
            echo "<p><em>Warning</em>: Someone might of been brute forcing your account.</p>";
            echo "<p>Number of login attempts: <em>{$failed_login}</em>.<br />Last login attempt was at: <em>${last_login}</em>.</p>";
        }

        // Reset bad login count
        $data = $db->prepare( 'UPDATE users SET failed_login = "0" WHERE user = (:user) LIMIT 1;' );
        $data->bindParam( ':user', $user, PDO::PARAM_STR );
        $data->execute();
    }
    else {
        // Login failed
        sleep( rand( 2, 4 ) );

        // Give the user some feedback
        echo "<pre><br />Username and/or password incorrect.<br /><br/>Alternative, the account has been locked because of too many failed logins.<br />If this is the case, <em>please try again in {$lockout_time} minutes</em>.</pre>";

        // Update bad login count
        $data = $db->prepare( 'UPDATE users SET failed_login = (failed_login + 1) WHERE user = (:user) LIMIT 1;' );
        $data->bindParam( ':user', $user, PDO::PARAM_STR );
        $data->execute();
    }

    // Set the last login time
    $data = $db->prepare( 'UPDATE users SET last_login = now() WHERE user = (:user) LIMIT 1;' );
    $data->bindParam( ':user', $user, PDO::PARAM_STR );
    $data->execute();
}

// Generate Anti-CSRF token
generateSessionToken();

?>
```
