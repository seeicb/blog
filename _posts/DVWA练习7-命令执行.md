title: DVWA练习7-命令执行
date: 2016-03-21 22:43:24
categories: 安全
tags: Web安全

---
### 一、命令执行
#### 1.简介
命令执行漏洞指攻击者可以随意执行系统命令，属于代码执行。在Web程序中经常提供一些执行系统命令的函数，当用户可以输入任意命令，没有过滤时，就会发生命令执行漏洞。
<!--more-->
#### 2.成因
很多语言中都有执行系统命令的函数。
如php中 system(),shell_exec(),exec(),passthru(),passthru(),pvntl_exec(),popen(),proc_open()。java中的runtime类等。
还有一些框架命令执行，如struts2代码执行漏洞等。
很多时候利用'&&'，'||'，'|'，'&'，';'，'$','`'等。

#### 3.防御  
尽量不要使用执行系统命令函数。    
在参数进入执行命令之前，加强过滤    
使用escapeshellcmd(),escapeshellarg()函数过过滤    
使用参数白名单方式  


### 二、DVWA练习
#### 1.Low
此处本该输入IP进行ping测试，可以输入114.114.114.114&&net user来查看用户名
“&&”是命令连接。
如图
![](http://7xo8y2.com1.z0.glb.clouddn.com/dvwacomex0001.png)
查看源码：
```php

<?php
if( isset( $_POST[ 'Submit' ]  ) ) {
    $target = $_REQUEST[ 'ip' ];
    // 选择操作系统
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // php_uname — 返回运行 PHP 的系统的有关信息
        //Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }
    // 返回结果
    echo "<pre>{$cmd}</pre>";
}
?>
```
在这里没有任何过滤，所以可以执行各种命令。
#### 2.Medium
再次使用114.114.114.114&&net user,返回错误
"Ping 请求找不到主机 114.114.114.114net。请检查该名称，然后重试。"
推测是过滤了&&，可以使用"|"管道符来执行，如图。
![](http://7xo8y2.com1.z0.glb.clouddn.com/dvwacomex0003.png)
查看源码：
```php
<?php
if( isset( $_POST[ 'Submit' ]  ) ) {
    $target = $_REQUEST[ 'ip' ];
    // 过滤黑名单'&&',';'
    $substitutions = array(
        '&&' => '',
        ';'  => '',
    );
    // 替换黑名单
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );
    // 确认系统，执行命令
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }
    echo "<pre>{$cmd}</pre>";
}
?>
```
虽然进行了过滤，但是并不严格，还是可以绕过。

#### 3.High
查看源码：
```php
<?php
if( isset( $_POST[ 'Submit' ]  ) ) {
    $target = trim($_REQUEST[ 'ip' ]);
    // 设置黑名单
    $substitutions = array(
        '&'  => '',
        ';'  => '',
        '| ' => '',
        '-'  => '',
        '$'  => '',
        '('  => '',
        ')'  => '',
        '`'  => '',
        '||' => '',
    );
    // 替换黑名单
    $target = str_replace( array_keys( $substitutions ), $substitutions, $target );
    if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
        // Windows
        $cmd = shell_exec( 'ping  ' . $target );
    }
    else {
        // *nix
        $cmd = shell_exec( 'ping  -c 4 ' . $target );
    }
    echo "<pre>{$cmd}</pre>";
}
?>
```
~~加强了过滤名单，帮助上提示trim()函数，暂时还不知道如何突破。~~
更新：仔细观察黑名单中第三行`'| ' => ''`这里存在一个空格，所以实际上没有过滤|，输入`114.114.114.114&net user`即可。

#### 4.Impossible
Impossible级别里分离了输入数据，将其保存到数组，对每个数组的值进行判断，是否为数字，是则执行命令，错误不能执行。
查看源码：
```php
<?php
if( isset( $_POST[ 'Submit' ]  ) ) {
    // CSRF防御
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
    $target = $_REQUEST[ 'ip' ];
    $target = stripslashes( $target );
    //stripslashes() 函数删除由 addslashes() 函数添加的反斜杠。
    // explode() 函数把字符串打散为数组。保存到$octet
    $octet = explode( ".", $target );
    // 检查$octet数组是否为数字
    if( ( is_numeric( $octet[0] ) ) && ( is_numeric( $octet[1] ) ) && ( is_numeric( $octet[2] ) ) && ( is_numeric( $octet[3] ) ) && ( sizeof( $octet ) == 4 ) ) {
        // 合成IP地址
        $target = $octet[0] . '.' . $octet[1] . '.' . $octet[2] . '.' . $octet[3];
        if( stristr( php_uname( 's' ), 'Windows NT' ) ) {
            // Windows
            $cmd = shell_exec( 'ping  ' . $target );
        }
        else {
            // *nix
            $cmd = shell_exec( 'ping  -c 4 ' . $target );
        }
        echo "<pre>{$cmd}</pre>";
    }
    else {
        echo '<pre>ERROR: You have entered an invalid IP.</pre>';
    }
}
// Generate Anti-CSRF token
generateSessionToken();
?>
```
