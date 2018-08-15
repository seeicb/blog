title: Windows和Linux双系统引导设置
date: 2016-03-10 21:42:16
categories: Linux  
tags: Linux
---
下载ubuntu选择UEFI安装后，与Windows10共存，但是启动时没有Linux,需要设置一下引导。
打开命令提示符(管理员)
运行下列命令将grub64.efi设置为启动引导程序： bcdedit /set {bootmgr} path \EFI\ubuntu\grubx64.efi
重启后会进入GRUB菜单，如果没有成功，输入bcdedit /set {bootmgr} path \EFI\ubuntu\shimx64.efi
如果想设置为Windows，可以输入 bcdedit /set {bootmgr} path \EFI\Microsoft\grubx64.efi
