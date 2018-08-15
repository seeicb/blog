---
title: python爬虫学习
date: 2016-04-17 21:31:12
categories: Python
tags: 爬虫
---
##  
最近开始学python,按照慕课网 http://www.imooc.com/learn/563  上的教程，写了个简单的爬虫，记录一下。  
[代码地址](https://github.com/seeicb/my_code/tree/master/spider_baike)

<!--more-->
## 模块
### spider_main     爬虫主程序
  `__init__`      初始化  
  `craw` 爬取程序  
### url_manager     url管理器
  `__init__` 初始化  
  `add_new_url`   增加url  
  `add_new_urls`  增加urls  
### html_parser     html解析器
  `parse`       解析函数  
  `_get_new_urls` 得到urls  
  `_get_new_data` 得到html数据  

### html_download html下载器
  `download`      下载函数  
### html_out   html输出器
  `__init__`      初始化  
  `collect_data`  收集数据  
  `output_html`   输出函数

使用了urllib BeautifulSoup4  re模块


## 流程
定义开始页面，将root(开始页面)加入到self.new_urls中。  
　　判断self.new_urls是否为空，不为空就继续。  
　　　　通过url管理器_get_new_urls函数在new_urls中取最后一个,加入到old_urls中,返回值赋给new_url  
　　　　下载器下载new_url页面内容  
　　　　解析器解析new_url页面。取得所有符合条件的url，加入到new_urls中。取得页面中的数据，即页面url，标题，简介。  
　　　　返回new_urls和new_data  
　　　　通过url管理器，将new_urls加入到self.new_urls(这两个不一样)。  
　　　　通过html输出器，将new_data输出。  
　　　　对记录数加1，并判断是否取得足够数据，不满则继续循环。  
　　此时将会再次判断self.new_urls是否为空，然后从self.new_urls取一个url继续以上内容  
