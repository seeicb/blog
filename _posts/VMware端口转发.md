title: VMware端口转发
date: 2015-11-15 22:14:30
categories: Linux
tags: Linux

---
因为在ubuntu中使用虚拟机搭建了一个web服务器，在主机上可以使用host-only访问，但是其它的电脑无法访问，就想到了端口转发功能。
在linux 下，vmware的网络编辑没有这个设置，需要在配置文件中设置。
配置文件位于: /etc/vmware/vmnet8/nat/nat.conf
首先备份这个文件，免得设置错误，无法上网。
<!--more-->
在文件后面可以看到
	[incomingtcp] //这里是TCP连接的转发
	# Use these with care - anyone can enter into your VM through these...
	# The format and example are as follows:
	#<external port number> = <VM's IP address>:<VM's port number>
	#8080 = 172.16.3.128:80
	[incomingudp] //这里是UDP连接的转发
	# UDP port forwarding example
	#6000 = 172.16.3.0:6001
举个例子：
要转发80端口，可以在tcp下设置为
8080 = 172.16.3.128:80
这里8080是物理机的端口，ip是虚拟机的ip，80是虚拟机的端口。
设置好后，要重启服务。

	/usr/bin/vmware-networks stop
	/usr/bin/vmware-networks start
