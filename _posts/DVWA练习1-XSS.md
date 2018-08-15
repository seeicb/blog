title: DVWA练习1-XSS
date: 2015-12-02 23:28:56
categories: 安全
tags: [Web安全,XSS]

---
### 一、XSS
#### 1.XSS简介  
 XSS即跨站脚本攻击，恶意攻击者往Web页面里插入恶意html代码，当用户浏览该页之时，嵌入其中Web里面的html代码会被执行，从而达到恶意攻击用户的特殊目的。
XSS通常分为三类：反射型XSS 存储型XSS DOM型XSS ，还有flash xss。  
<!--more-->
#### 2.XSS危害：
1.最常用的盗取cookie
2.劫持用户会话
3.网页挂马等等。
#### 3.XSS攻击平台：
1.http://webxss.cn/   
2.BeEF  
3.还有一些XSS扫描工具。
#### 4.XSS构造
1.wooyunBypassxss过滤的测试方法http://drops.wooyun.org/tips/845  
2.利用字符编码  
3.绕过长度限制  
#### 5.XSS防御
1.httponly  
2.对输入的数据进行检查，使用一些编码函数，如HtmlEncode,encodeJavaScript
### 二、DVWA练习
#### 1.low
安全级别为low时，没有任何防护，直接输入
  ``<script>alert("xxs")</script>``
但是在chrome下无法实现，推测是浏览器拦截了，在firefox下可以实现。  
查看源码如下：
```php
<?php
// name是否存在，并且name不为空
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	//返回name,此处没有任何过滤
	$html .= '<pre>Hello ' . $_GET[ 'name' ] . '</pre>';
}
?>
```
#### 2.Medium
当输入``<script>alert("xxs")</script>``
发现输出为Hello alert("xxs")
查看源码如下：
```php
<?php
// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	// Get input
	$name = str_replace( '<script>', '', $_GET[ 'name' ] );

	// Feedback for end user
	$html .= "<pre>Hello ${name}</pre>";
}
?>
```
str_replace() 函数以其他字符替换字符串中的一些字符（区分大小写）,在此将 ``<script> ``修改大小写即可。``<SCript>alert("xxs")</SCript>``
#### 3.High
当输入``<script>alert("xxs")</script>``
输出 Hello > 。
查看源码
```php
<?php
// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	// Get input
	$name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] );

	// Feedback for end user
	$html .= "<pre>Hello ${name}</pre>";
}
?>
```
preg_replace执行一个正则表达式的搜索和替换.在此替换了``<script>``中的内容，所以需要输入中不包含``<script>``
可以使用img标签来绕过。
输入：``<img src=1 onerror=alert(1)>  ``

#### 4.Impossible
```php
<?php
// Is there any input?
if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {
	// Check Anti-CSRF token
	checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' );
	// Get input
	$name = htmlspecialchars( $_GET[ 'name' ] );
	// Feedback for end user
	$html .= "<pre>Hello ${name}</pre>";
}
// Generate Anti-CSRF token
generateSessionToken();
?>
```
htmlspecialchars函数把预定义的字符 "<" （小于）和 ">" （大于）转换为 HTML 实体：
没有办法过滤
