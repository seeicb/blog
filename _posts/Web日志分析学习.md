---
title: Web日志分析学习
comments: false
date: 2016-08-09 23:50:32
categories: 安全
tags: Web安全
---
### Apache日志分析

Apache默认日志存在于安装目录下的logs文件中。
<!--more-->

其记录基本为：
```
127.0.0.1 - - [09/Jul/2016:16:37:49 +0800] "GET / HTTP/1.1" 302 - "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36"

127.0.0.1 - - [09/Jul/2016:16:37:54 +0800] "GET /dashboard/images/xampp-logo.svg HTTP/1.1" 200 5427 "http://127.0.0.1/dashboard/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36"
```
其格式为：  
**远程IP**  即访问网站的IP地址，此处为本机地址。    
**访问者的标示** 此处一般为空白，使用`-`占位符来代替。  
**身份验证** 用于记录浏览者进行身份验证时提供的名字，空白时用`-`占位符来代替。  
**时间** 用来记录访问时间  
**请求信息** 其格式为“METHOD RESOURCE PROTOCOL”，即“方法 资源 协议”，在本例中第一项为GET方法、访问根目录、使用HTTP1.1  
**状态码** 此处为http的状态码  
**字节数** 此处记录发送给客户端的字节数  
**HTTP Referer**  告诉服务器是从哪个页面跳转过来的，空白时用`-`占位符来代替。  
**浏览器标识** 此处记录浏览器的信息  


一般而言，要注意哪些访问敏感目录的请求，以及请求中带有攻击行为的日志。经常能够看到网络上的各种无脑扫描弱口令，敏感目录的IP。  
### Tomcat日志分析
Tomcat日志一般存放在安装目录下的logs文件中。
访问日志的配置位于server.xml，  
```
<Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
       prefix="localhost_access_log." suffix=".txt"
       pattern="%h %l %u %t &quot;%r&quot; %s %b" />

```
pattern表示日志生产的格式。  
```
* %h 访问的用户IP地址
* %l 访问逻辑用户名，通常返回'-'
* %u 访问验证用户名，通常返回'-'
* %t 访问日时
* %r 访问的方式(post或者是get)，访问的资源和使用的http协议版本
* %s 访问返回的http状态
* %b 访问资源返回的流量
* %T 访问所使用的时间
```

日志记录样例为
```
127.0.0.1 - - [11/Aug/2016:21:33:22 +0800] "GET /manager/html HTTP/1.1" 401 2538
```


对于Tomcat，要特别注意那些频繁访问`/manager/html`，状态码为401的IP，因为这很有可能是在尝试爆破Tomcat管理账户密码。

### Nginx
nginx日志默认位置为nginx安装目录下logs文件中。分别为访问日志和错误日志。    
访问默认日志配置为：    
```
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;
```
日志示例：    
```
192.168.80.1 - - [15/Aug/2016:11:35:58 +0800] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.82 Safari/537.36"
```
其格式分别为：    
$remote_addr 远程IP地址  
\- （占位符）  
$remote_user 远程客户端用户名（没有使用-代替）  
[$time_local] 访问时间与时区  
$request 请求方法与协议  
$status HTTP状态码  
$body_bytes_sent 发送给客户端内容大小  
$http_referer 从那个页面访问过来  
$http_user_agent 浏览器UA  
$http_x_forwarded_for 客户端的真实ip。    

### IIS
IIS7.5日志默认存放在`%SystemDrive%\inetpub\logs\LogFiles`    
旧版本的IIS默认默认位置为`%systemroot%\system32\logfiles\w3svc1\`    
IIS日志格式说明：    

>以下内容引用自Windows帮助文件  <html</p><div id="sectionSection0" class="section"><content xmlns="http://ddue.schemas.microsoft.com/authoring/2003/5"><p xmlns=""><b></b></p><table xmlns=""><tr><th colspan="1">元素名称</th><th colspan="1">描述</th></tr><tr><td colspan="1"><p><b>日期 (date)</b></p></td><td colspan="1"><p>记录发出请求的日期。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>时间 (time)</b></p></td><td colspan="1"><p>记录发出请求的时间（协调世界时 (UTC)）。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>客户端 IP 地址 (c-ip)</b></p></td><td colspan="1"><p>记录发出请求的客户端的 IP 地址。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>用户名 (cs-username)</b></p></td><td colspan="1"><p>记录访问服务器的已经过验证用户的名称。匿名用户用连字符来表示。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>服务名 (s-sitename)</b></p></td><td colspan="1"><p>记录当记录事件时运行于客户端上的 Internet 服务的名称和实例的编号。</p></td></tr><tr><td colspan="1"><p><b>服务器名称 (s-computername)</b></p></td><td colspan="1"><p>记录生成日志文件项的服务器的名称。</p></td></tr><tr><td colspan="1"><p><b>服务器 IP 地址 (s-ip)</b></p></td><td colspan="1"><p>记录生成日志文件项的服务器的 IP 地址。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>服务器端口 (s-port)</b></p></td><td colspan="1"><p>记录为服务配置的服务器端口号。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>方法 (cs-method)</b></p></td><td colspan="1"><p>记录在请求中使用的 HTTP 方法，例如 <b>GET</b>。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>URI 资源 (cs-uri-stem)</b></p></td><td colspan="1"><p>记录作为操作目标的统一资源标识符 (URI)，例如 Default.htm。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>URI 查询 (cs-uri-query)</b></p></td><td colspan="1"><p>记录客户端尝试执行的查询（如果有）。只有动态页面需要 URI 查询。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>协议状态 (sc-status)</b></p></td><td colspan="1"><p>记录 HTTP 状态代码。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>协议子状态 (sc-substatus)</b></p></td><td colspan="1"><p>记录 HTTP 子状态代码。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>Win32 状态 (sc-win32-status)</b></p></td><td colspan="1"><p>记录 Windows 状态代码。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>发送的字节数 (sc-bytes)</b></p></td><td colspan="1"><p>记录服务器发送的字节数。</p></td></tr><tr><td colspan="1"><p><b>接收的字节数 (cs-bytes)</b></p></td><td colspan="1"><p>记录服务器接收的字节数。</p></td></tr><tr><td colspan="1"><p><b>所用时间 (time-taken)</b></p></td><td colspan="1"><p>记录操作所花费的时间（毫秒）。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>协议版本 (cs-version)</b></p></td><td colspan="1"><p>记录客户端使用的协议版本（HTTP 或 FTP）。</p></td></tr><tr><td colspan="1"><p><b>主机 (cs-host)</b></p></td><td colspan="1"><p>记录主机头名称（如果有）。</p><table class="alertTable" cellspacing="0" cellpadding="0"><tr><td class="imgCell"></td><td class="txtCell">注意 </td></tr><tr><td class="imgCell"></td><td class="alertCell"><p>为网站配置的主机名可能会以不同的方式出现在日志文件中，原因是 HTTP.sys 使用 Punycode 编码格式来记录主机名。</p></td></tr></table></td></tr><tr><td colspan="1"><p><b>用户代理 (cs(User-Agent))</b></p></td><td colspan="1"><p>记录请求所来自的浏览器。默认情况下为选定状态。</p></td></tr><tr><td colspan="1"><p><b>Cookie (cs(Cookie))</b></p></td><td colspan="1"><p>记录发送或接收的 Cookie 内容（如果有）。</p></td></tr><tr><td colspan="1"><p><b>引用站点(cs(Referer)</b><b>)</b></p></td><td colspan="1"><p>记录用户上次访问的网站。此站点提供与当前站点的链接。</p></td></tr></table></content></div><h1 class="heading"><span id="seeAlsoNoToggle">


对于默认的记录字段如图：
![](http://7xo8y2.com1.z0.glb.clouddn.com/%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90IIS%E5%AD%97%E6%AE%B5.png)
![](http://7xo8y2.com1.z0.glb.clouddn.com/%E6%97%A5%E5%BF%97%E5%88%86%E6%9E%90IIS%E5%AD%97%E6%AE%B52.png)

IIS日志示例：  
```
#Fields: date time s-ip cs-method cs-uri-stem cs-uri-query s-port cs-username c-ip cs(User-Agent) sc-status sc-substatus sc-win32-status time-taken
2016-08-15 02:40:33 192.168.80.130 GET /welcome.png - 80 - 192.168.80.1 Mozilla/5.0+(Windows+NT+10.0;+Win64;+x64)+AppleWebKit/537.36+(KHTML,+like+Gecko)+Chrome/52.0.2743.82+Safari/537.36 200 0 0 31
```


### 参考

http://blog.163.com/lgh_2002/blog/static/44017526201002211543558/  
http://twb.iteye.com/blog/182100  
http://blog.chinaunix.net/uid-26153587-id-4090945.html  
