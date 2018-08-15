title: DVWA练习5-文件包含漏洞
date: 2016-03-11 13:32:17
categories: 安全
tags: Web安全
---

### 一、文件包含漏洞
#### 1.简介
如果允许客户端用户输入控制动态包含在服务器端的文件，会导致恶意代码的执行及敏感信息泄露，主要包括本地文件包含和远程文件包含两种形式。
PHP: include(),include_once(),require(),require_once(),fopen(),readfile()  
JSP/Servlet: ava.io.File(),java.io.FileReader()  
ASP: include file,include virtual  
<!--more-->
#### 2.利用
使用"../"遍历目录,读取敏感信息  
%00截断目录  
使用PHP封装协议  
包含Apache日志文件  
目录长度限制  
参考：http://wiki.wooyun.org/web:lfi http://drops.wooyun.org/tips/3827
#### 3.防御
路径限制，PHP设置open_basedir  
严格验证文件，是否为外部可控。


### 二、DVWA练习
#### 1.Low
点击file1，url为"http://127.0.0.1/dvwa/vulnerabilities/fi/?page=file1.php"
直接修改url "http://127.0.0.1/dvwa/vulnerabilities/fi/?page=../../phpinfo.php" 可得到phpinfo信息。
如果开启allow_url_include则可以使用远程包含。
查看源码没有任何防护。
```php
<?php
// The page we wish to display
$file = $_GET[ 'page' ];
?>
```
#### 2.Medium
中级的使用../../vulnerabilities/fi/file1.php 提示没有文件,错误信息如下。
```
Warning: include(vulnerabilities/fi/file1.php): failed to open stream: No such file or directory in F:\xampp\htdocs\DVWA\vulnerabilities\fi\index.php on line 36
```
可见是过滤了"../"
查看源码：
```php
<?php

// The page we wish to display
$file = $_GET[ 'page' ];
// Input validation
$file = str_replace( array( "http://", "https://" ), "", $file );
$file = str_replace( array( "../", "..\"" ), "", $file );
?>
```
使用 str_replace过滤，在SQL注入medium中使用此方法过滤。绕过方法为"..././..././phpinfo.php"，在过滤了"../"后
即为"../../phpinfo.php"
远程包含也可以改变大小写，如"HTTp://"。
#### 3.High
查看源码:
```php
<?php
// The page we wish to display
$file = $_GET[ 'page' ];

// Input validation
//fnmatch() 函数根据指定的模式来匹配文件名或字符串。目前该函数无法在 Windows 或其它非 POSIX 兼容的系统上使用。如果$file不是include.php。不正确
if( !fnmatch( "file*", $file ) && $file != "include.php" ) {
	// This isn't the page we want!
	echo "ERROR: File not found!";
	exit;
}
?>
```
在这里限制了文件名来防御。
更新：要求文件名必须以file为开头，所以可以使用file协议来打开文件。例如
`[http://127.0.0.1/dvwa/vulnerabilities/fi/page=file:///E:/xampp/htdocs/dvwa/php.ini`

#### 4.Impossible
源码：
```php
<?php
// The page we wish to display
$file = $_GET[ 'page' ];
// Only allow include.php or file{1..3}.php
if( $file != "include.php" && $file != "file1.php" && $file != "file2.php" && $file != "file3.php" ) {
	// This isn't the page we want!
	echo "ERROR: File not found!";
	exit;
}
?>
```
