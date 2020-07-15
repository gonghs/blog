---
title: Windows10基于WSL2的Ubuntu子系统安装
date: 2020-07-09 18:35:22
categories: 
- windows
tags:
- ubuntu
description: WSL是微软为了方便开发者编程，在Windows10中内嵌的Linux子系统，最近WSL2作为标准组件成为Windows10 2004版本的一部分，比起一代提升了文件系统的I/O性能和与Linux的兼容性。
---
## 启用windows功能

控制面板 -> 程序 -> 程序和功能 -> 启用或关闭Windows功能或Win + R运行control appwiz.cpl指令

![启用或关闭windows功能](https://gitee.com/gonghs/image/raw/master/img/20200628103732.png)

在功能列表中找到**适用于Linux的Windows子系统**和**虚拟机平台**两项启用。

![启用功能](https://gitee.com/gonghs/image/raw/master/img/20200628104633.png)

## 安装WSL1

WSL1依赖于上步骤**适用于Linux的Windows子系统**功能。

**使用管理员身份**运行Powershell并执行指令：

```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```

![安装WSL1](https://gitee.com/gonghs/image/raw/master/img/20200628105140.png)

如果只安装WSL1此时应该重启，否则更新至WSL2再重启。

## 更新WSL2

更新WSL2需要**Windows10版本2004且内部版本高于19041**。

版本信息可以通过Win+R运行winver确认（下图版本不可用，需要更新）。

![版本信息](https://gitee.com/gonghs/image/raw/master/img/20200628110225.png)

或在Cmd中使用ver命令确认。

![ver指令](https://gitee.com/gonghs/image/raw/master/img/20200628110352.png)

版本或内部版本不满足需求使用Win+S搜索**更新设置**进行系统更新。

![20200628110818](https://gitee.com/gonghs/image/raw/master/img/20200628114535.png)

如果版本非2004版本可能需要访问下载[Windows更新助手](https://www.microsoft.com/zh-cn/software-download/windows10)升级至2004版本。

![更新工具](https://gitee.com/gonghs/image/raw/master/img/20200628115058.png)

WSL2依赖于**虚拟机平台**功能。

使用管理员身份运行Powershell并执行指令：

```
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

![更新WSL2](https://gitee.com/gonghs/image/raw/master/img/20200707181605.png)

此时重启再进行后续操作。

##  安装Ubuntu

安装新的 Ubuntu时，先将 WSL 2 设置为默认：

```
wsl --set-default-version 2
```

若提示：

![需要更新内核](https://gitee.com/gonghs/image/raw/master/img/20200707181745.png)

则需要进行[linux内核升级](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)（若下载太慢可在公众号后台回复wsl网盘链接下载）。

升级完毕后执行指令将提示：

![image-20200707235147861](https://gitee.com/gonghs/image/raw/master/img/20200707235154.png)

若需要修改已有系统的wsl版本，可以执行：

```
-- 查运行系统名称
wsl -l -v
-- 设置指定名称系统版本
wsl --set-version Ubuntu-20.04 2
```

在微软商店搜索Ubuntu（如果使用的是WSL1，[建议安装18.04版本](https://discourse.ubuntu.com/t/ubuntu-20-04-and-wsl-1/15291)）

![image-20200707143948800](https://gitee.com/gonghs/image/raw/master/img/20200707143949.png)

安装完毕直接启动并设置初始用户名密码便可以进入子系统：

![安装完毕](https://gitee.com/gonghs/image/raw/master/img/20200708000602.png)

## 附：可以使用Windows Terminal打开系统

在Windows Terminal中选择Ubuntu打开：

![使用Windows Terminal打开](https://gitee.com/gonghs/image/raw/master/img/20200709145140.png)