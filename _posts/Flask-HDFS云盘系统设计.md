---
title: Flask+HDFS云盘系统设计
date: 2017-07-10 21:22:39
categories: Python
tags: Flask
---


通过学习Flask，Hadoop等开发了一个简单的云盘系统，记录一下。    
项目地址：https://github.com/seeicb/Flask-Disk  
<!--more-->
### 技术选型

前端：Bootstrap JQuery sweetalert(弹窗插件) DataTables(表格插件) Webuploader(上传插件)  
后端：Flask  
数据库： [SQLAlchemy](http://baike.baidu.com/item/SQLAlchemy)  
文件存储：Hadoop  
部署：Nginx+Uwsgi+supervisor  

### 数据库设计

数据库设计：数据库设计采用SQLAlchemy进行模型设计，用熟了之后相比原生的SQL语句还是比较方便的，整个云盘分为四张表，分别为文件表，用户表，分享表，删除表。ER图如下：  

![E-R图](http://7xo8y2.com1.z0.glb.clouddn.com/17-7-13/36820088.jpg)

### 功能需求

用户管理：注册、登录、个人主页、资料编辑和修改密码  

文件管理：文件上传下载、文件公开分享和密码分享、删除文件、恢复文件和重命名文件  

管理员后台：查看用户个人信息、锁定普通用户、修改用户密码  （后台功能比较渣）  

### 重点流程

用户登录退出：

文件上传和下载：

文件的上传与下载依赖于Python hdfs模块，安装与使用代码：

```python
pip install hdfs
from hdfs import Client
HDFS_client = Client("http://127.0.0.1:50070")
```

上传文件时，前端通过Webuploader插件实现，后台接受到文件后先将文件属性存入数据库，再将文件通过IO流传入hadoop中，上传流程图如下：

![上传流程图](http://7xo8y2.com1.z0.glb.clouddn.com/17-7-13/90041101.jpg)

文件下载：

```python
def download(id):
    # 根据文件ID获取文件对象
    fileobj = FileTable.query.filter_by(file_id=id, user_id=current_user.get_id(), is_recycle=False).first_or_404()
    path = fileobj.hdfs_path + fileobj.hdfs_filename
    # 得到文件真实路径
    filename = fileobj.filename
    # 读取文件内容
    with hdfs_client.read(path) as reader:
        buf = reader.read()
    response = make_response(buf)
    response.headers.add('Content-Disposition', 'attachment', filename=filename.encode())
    # 返回给浏览器下载文件
    return response

```

文件删除：

删除文件会先将文件移动到回收站，主要是根据Hadoop回收站机制，在HDFS里,删除文件时，会先将文件移动到回收站，而不会真的删除，回收站中的文件可以还原，同时会在配置文件中设置一个时间长度，当超过这个时间长度后，HDFS会自动清除文件。

文件分享：

文件分享分为公开分享和密码分享。文件分享链接是根据文件ID，HDFS文件名，UNXI时间戳生成的MD5值作为唯一链接，密码分享时同时会生成随机4位密码作为分享密码。当访问文件时，会检查分享类型，如果是密码分享就需要输入正确密码。公开分享可以直接访问。

### 系统部署

由于资源有限，整个系统是安装在一台服务器上的，Hadoop采用伪分布式安装，使用Python3开发，部署方案采用Nginx+Uwsgi+supervisor。简单的来说就是最外层使用Nginx服务器，中间使用Uwsgi作为WSGI接口，最后是flask应用。
而supervisor作为进程管理工具用来启动，重启，关闭整个服务。关于这方面的部署都可以在网上搜得到。

### 总体效果

系统首页

![首页](http://7xo8y2.com1.z0.glb.clouddn.com/17-7-13/85436856.jpg)

文件详细页

![文件详细页](http://7xo8y2.com1.z0.glb.clouddn.com/17-7-13/3140002.jpg)

上传页

![上传页](http://7xo8y2.com1.z0.glb.clouddn.com/17-7-13/54958858.jpg)











​           
