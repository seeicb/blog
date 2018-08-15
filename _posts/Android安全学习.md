---
title: Android安全学习
date: 2017-09-18 20:58:00
categories: 安全
tags: [Android]
---


最近学习了一些Android方面的安全知识，记录一下。

Android是基于Linux内核的操作系统。Android应用的问题主要存在于四大组件，即活动（Activity），服务（Service），广播接收器（Boardcast Reciver）和内容供应器（Content Provider）。Android安全的机制包括权限管理、签名认证、沙箱机制、访问控制、进程通信机制、内存管理机制等。
Android安全包括了好几个方面，包括内核安全，系统框架层安全，应用安全等。个人学习的主要方向是在应用层，本文写的比较杂，也比较基础。
<!--more-->
## APK文件分析：

*APK*是AndroidPackage的缩写，即Android安装包。APK文件可以通过解压工具进行解压，可以查看apk中都包含了那些文件。

通常情况下都包括以下文件:

* Classes.dex (Dalvik 可执行文件)
* AndroidManifest.xml (项目清单文件)
* resources.arsc (编译后的二进制资源文件)
* META-INF (签名信息)
* res (资源文件)
* assets (静态文件)
* lib (native库文件)

其中，AndroidManifest.xml 包括了应用的包名、版本号、app权限声明、是否允许调试、是否允许备份、是否有共享用户组、可被导出的组件等。对于可导出的组件需要重点关注。  
Classes.dex是apk的核心文件，是由 Java 字节码转换的 Dalvik 字节码， 可以通过反编译工具查看java源代码。
[APK文件结构和安装过程](http://blog.csdn.net/bupt073114/article/details/42298337)

## APK生成过程
![APK生成过程](http://7xo8y2.com1.z0.glb.clouddn.com/17-9-14/54392223.jpg)
如图，整个APK生成过程如下。
1、打包资源文件，生成R.java文件
2、处理AIDL文件，生成相应.java 文件
3、编译Java文件，生成对应的.class文件
4、把.class文件转化成Davik VM支持的.dex文件
5、打包生成未签名的.apk文件
6、对apk文件进行签名
7、对签名后的.apk文件进行对齐处理

[APK打包过程·](http://blog.csdn.net/jason0539/article/details/44917745)

#### 签名验证

app安装时必须要有数字签名才能够进行安装，在安装时会校验包名和签名信息，当应用升级时，会校验签名是否相同，否则会安装失败。只有包名和签名相同才会被认为是同一个应用。
数字签名采用非对称加密算法生成。可以使用jarsigner和signapk工具进行签名。
签名信息在META-INF文件下，包括CERT.RSA，CERT.SF，MANIFEST.MF。
其中MANIFEST.MF列出了apk的所有文件和对应SHA1哈希值文件base64后的值。 
CERT.SF是对MANIFEST.MF文件中每条信息再次SHA1并base64编码后的值。  
CERT.RSA包含了CERT.SF文件的数字签名和公钥信息。  
[Android中签名原理和安全性分析](http://blog.csdn.net/lostinai/article/details/54694564)

## 工具

#### adb  

adb的全称为Android Debug Bridge，就是起到调试桥的作用。通过adb可以对应用程序进行调试。
常用命令如下:

```bash
adb devices  
#列举当前连接的调试设备
adb logcat
#打印log信息
adb install/uninstall
#安装卸载apk
adb shell
#进入设备的Linux Shell环境
adb push/pull
#传输文件到调试设备/拷贝文件到PC
```

[adb命令参考](http://www.jianshu.com/p/5980c8c282ef)

#### drozer

drozer是一款Android APP安全评估工具，在APP安全测试方面非常有用。drozer需要安装手机agent端和PC端。[官网链接](https://labs.mwrinfosecurity.com/tools/drozer/)

使用时需要在手机端打开Server，如图![](http://7xo8y2.com1.z0.glb.clouddn.com/17-9-18/11479383.jpg)


常用命令:

```bash
adb forward tcp:31415 tcp:31415
#PC端端口转发
drozer console connect 
#进入drozer
run app.package.list -f example
##列出所有包名包含xxx的应用包
run app.package.info -a com.example
##列出com.example包的详细信息
run app.package.attacksurface com.example
#app攻击面分析,包括可导出的组件，是否可以调试
run app.activity.info -a com.example
#显示一个包的导出的activity组件详情
run app.service.info -a com.example
#显示一个包的导出的service组件详情
run app.broadcast.info -a com.example
#显示一个包的导出的broadcast组件详情
run app.provider.info -a com.example
#显示一个包的导出的provider组件详情

```


[drozer进行Android渗透测试](http://blog.csdn.net/lostinai/article/details/48999713)

## 逆向分析
#### [dex2jar](https://github.com/pxb1988/dex2jar)+[jd-gui](http://jd.benow.ca/)	

dex2jar可以将apk反编译成java源码。
使用命令

```
#  d2j-dex2jar.bat example.apk
dex2jar example.apk -> .\example-dex2jar.jar
```

然后可以在当前目录下得到example-dex2jar.jar
再使用jd-gui打开example-dex2jar.jar，即可查看java源码。

#### [APKIDE](https://www.52pojie.cn/thread-399571-1-1.html)

APKIDE是一款非常强大的Android逆向工具和环境，集成了以上所说的多种工具。
使用也十分简单。下载后打开软件，配置好java环境，直接左上角打开APK文件，即可自动反编译。

#### 抓包工具

对于Android网络抓包工具主要为burpsuite和fiddler。抓包时需要手机和PC处于一个网络下。配置完成后就可以浏览手机的具体请求。具体配置过程网上很多。

<!--有些包抓不到。协议不对，加密RSA  AES加密-->

## APP检测

#### Android漏洞检测平台

[360显微镜](http://appscan.360.cn/)
[阿里聚安全](http://jaq.alibaba.com/)
[腾讯金刚](http://service.security.tencent.com/kingkong)
[梆梆安全](https://dev.bangcle.com/Apps/index)

#### 渗透测试点

所有的APP数据可以在/data/data 下查看到，但是需要root权限。可以通过adb shell或者rootexplorer应用进行查看是否包含用户密码，手机号等敏感信息。

使用adb中的logcat查看是否有敏感信息泄露。

使用drozer测试APP攻击面，Activity越权，拒绝服务攻击等漏洞。

上传apk至在在线扫描器进行扫描。

使用burpsuite等抓包工具进行测试，此时就如同普通Web应用一样检测。

反编译应用，深入分析一些关键算法，加密流程等。



**参考信息**

[360安全知识库](http://appscan.360.cn/vulner/list/)


