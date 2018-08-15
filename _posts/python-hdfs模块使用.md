---
title: python hdfs模块使用
date: 2017-02-21 20:43:11
categories: Python
tags: HDFS

---


HDFS是hadoop分布式文件系统，HDFS中有两类节点。一类是NameNode，一类是DataNode。
其中NameNode是管理者，存储各种文件的元数据，SecondaryNameNode作为NameNode的冷备份。DataNode则是真正存储数据块，进行文件读写的地方。
更多内容参考[HDFS 原理、架构与特性介绍](http://www.uml.org.cn/sjjm/201309044.asp)
<!--more-->

python中对hdfs api封装的模块主要有hdfs,hdfs3,pytinyhdfs，PYHDFS等。这里选择hdfs比较简单清晰。

1.  安装
    `pip install hdfs`

2.  连接

    ```python
         >>> from hdfs import *
         >>> client=Client('http://192.168.71.109:50070')
    ```

         ​

         这里是在windwos上连接IP为192.168.71.109的安装有hadoop机器

3.  列出文件

    ```python
         >>> client.list('/')
         ['2017']
    ```

4.  新建文件夹

    ```python
         >>> client.makedirs('/a/b/c')
    ```

         可以递归创建文件夹，完成后不返回结果

5.  重命名/移动

    ```python
         >>> client.rename('/2017/02','/2017/03')
         >>> client.list('/2017')
         ['03']
    ```

         可以对文件进行重命名，移动操作，类似mv命令。函数原型`rename (hdfs_src_path, hdfs_dst_path)`

6.  删除

    ```python
         >>> client.delete('/2017',recursive=True)
         True
    ```

         recursive为递归删除，类似`rm -r`，函数原型`delete(hdfs_path, recursive=False)`

7.  写文件

    ```python
        >>> with open('D:/tmp/test.txt','rb') as f:
        ...     data=f.read()
        ...
        >>> client.write('/2017/02/test.txt',data=data)
    ```

       这里会报错，信息如下：

    ```python
      requests.exceptions.ConnectionError: HTTPConnectionPool(host='debian', port=50075): Max retries exceeded with url:
    /webhdfs/v1/2017/02/test.txt?op=CREATE&namenoderpcaddress=localhost:9000&overwrite=false (Caused by
    NewConnectionError('<requests.packages.urllib3.connection.HTTPConnection object at 0x03BB0D10>: Failed to establish a new connection: [Errno 11001] getaddrinfo
    failed',))
    ```

       这里`host='debian'`，`namenoderpcaddress=localhost:9000`，debian是安装hadoop的linux主机名，localhost:9000并不是namenode所在IP。原因是hdfs在读写操作中，namenode向客户端返回datanode时是主机名，而不是IP。所以需要在hosts文件中加入datanode主机名与IP的记录。

       ```
       #	127.0.0.1       localhost
       #	::1             localhost
       192.168.71.109 debian
       ```

       再来操作即可成功。

    ```python
       >>> client.write('/2017/02/test.txt',data=data)
       >>> client.list('/2017/02/')
       ['test.txt']
    ```


    函数原型：`write(hdfs_path, data=None, overwrite=False, permission=None, blocksize=None, replication=None, buffersize=None, append=False, encoding=None)`

    overwrite 当目标文件存在时，覆盖写入。

8.  读文件

    ```python
    >>> with client.read('/2017/02/test.txt') as f:
    ...     data=f.read()
    >>> print(data.decode())
    2017年2月21日13:11:37
    test
    ```

9.  上传

    ```python
       >>> client.upload('/2017/02/test.txt','D:/tmp/test.txt',overwrite=True)
       '/2017/02/test.txt'
    ```

10.  下载

    ```python
    >>> client.download('/2017/02/test.txt','D:/tmp/test2.txt')
    'D:\\tmp\\test2.txt'
    ```

**参考：[官方文档](https://hdfscli.readthedocs.io/en/latest/index.html)**
