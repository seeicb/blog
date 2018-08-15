---
title: mitmproxy-python-api使用
date: 2018-05-10 21:10:41
categories: Python
tags: mitmproxy

---

[mitmproxy](https://mitmproxy.org/)是一款使用python编写的中间人代理工具。包含了命令行界面、web界面和python api脚本三种方式进行代理抓包。其中python api可以用来编写自定义脚本，进行扩展，非常适合编写一些特定功能。

<!--more-->

简单的介绍下mitmproxy的Python api使用。

官网的简单例子，给http新增一个header头。

```
def response(flow):
    flow.response.headers["newheader"] = "foo"
```

运行方式 mitmdump -s add_header.py
这里flow.response.headers 获取http请求头信息，其它的还有

```
flow.request.host  http请求host
flow.request.method  请求方法
flow.request.scheme  请求协议
flow.request.url	 请求URL链接
flow.request.query   请求URL查询参数
flow.request.path    请求URL路径
flow.request.urlencoded_form  请求POST数据
flow.response.status_code  HTTP响应状态码
flow.response.headers    HTTP响应头信息
flow.response.get_text   HTTP响应内容
```

还有更多内容可以参考[官方文档](https://mitmproxy.readthedocs.io/en/v2.0.2/scripting/api.html)

以下代码为一个简单的脚本，作用是记录所有的流量，并保存到数据库中。

```
from mitmproxy.script import concurrent
import json,datetime
from sqlalchemy.databases import mysql
from sqlalchemy import Column, create_engine, Integer, Text,DateTime
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()


class Proxy(Base):
    __tablename__ = 'Proxy'
    id = Column(Integer, primary_key=True, autoincrement=True)
    str = Column(mysql.MSMediumText)
    time=Column(DateTime,default=datetime.datetime.utcnow)

engine = create_engine('mysql+pymysql://root:root@localhost:3306/test')
DBSession = sessionmaker(bind=engine)
Base.metadata.create_all(engine)
session = DBSession()

result = {}


@concurrent
def request(flow):
    domain= flow.request.host
    method= flow.request.method
    result['scheme'] = flow.request.scheme
    result['request_headers'] = {}
    for item in flow.request.headers:
        result['request_headers'][item] = flow.request.headers[item]
    url_path= flow.request.path
    result['get_data'] = parser_data(flow.request.query)
    
    result['post_data'] = parser_data(flow.request.urlencoded_form)  #
	

@concurrent
def response(flow):
    status_code = flow.response.status_code
    result['response_headers'] = {}
    for item in flow.response.headers:
        result['response_headers'][item] = flow.response.headers[item]
    result['response_content'] = flow.response.get_text()
    result_json = json.dumps(result)
    # print(result_json)

    #插入数据库
    new_url = Proxy(str=result_json)
    session.add(new_url)
    session.commit()
    # 关闭session:
    # session.close()


def parser_data(query):
    data = {}
    for key, value in query.items():
        data[key] = value
    return data
```

使用方法 `mitmdump -s script saveurl.py -p 8081` 

其中参数p为监听端口

另外如果直接在浏览器上挂代理的话会有很多其它无用的流量，推荐使用burpsuite的upstream功能。

如图，可以在这里自定义那些域名经过mitmproxy的代理，在渗透测试中可以有选择的将流量保存，便于日后分析。

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-5-13/73501947.jpg)

参考：

https://mitmproxy.readthedocs.io/en/v2.0.2/scripting/api.html