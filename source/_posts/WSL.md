---
title: WSL使用笔记
date: 2024-09-13 17:38:25
tags: WSL
categories: [WSL]
---

记录在windows中使用WSL技术出现的一系列问题

### 安装WSL

默认需要window10或11的系统，并且版本号有一定要求，可去 [官方](https://learn.microsoft.com/zh-cn/windows/wsl/install "WSL官方安装教程") 检查

#### 开启cpu虚拟化技术

使用WSL需要电脑cpu支持Hyper-V虚拟化技术，首先要在BIOS中开启对应选项，具体型号笔记本的选项不同，可参考  [在Windows 上启用虚拟化](https://prod.support.services.microsoft.com/zh-cn/windows/%E5%9C%A8-windows-%E4%B8%8A%E5%90%AF%E7%94%A8%E8%99%9A%E6%8B%9F%E5%8C%96-c5578302-6e43-4b4b-a449-8ced115f58e1 "在Windows 上启用虚拟化") 

<!-- more -->

#### Windows启用linux子系统

在Windows桌面任务栏的搜索框输入**启用或关闭Windows功能**，启用截图中圈选的功能

![avatar](WSL1.png)

但是实测发现，Windows中未开启Hyper-V和适用于Linux的Windows子系统，仅开启了虚拟机平台，WSL一样可用

#### WSL update

经过上述两步，可在终端控制台中输入`wsl --install`，自动安装所需工具，安装完成后输入`wsl -v`检查当前版本号和内核

如果此时提示命令行选项无效，可输入`wsl --update`，完成所需组件的更新

输入`wsl -v`最终显示如下信息
``` shell
WSL 版本： 2.2.4.0
内核版本： 5.15.153.1-2
WSLg 版本： 1.0.61
MSRDC 版本： 1.2.5326
Direct3D 版本： 1.611.1-81528511
DXCore 版本： 10.0.26091.1-240325-1447.ge-release
Windows 版本： 10.0.22631.4037
```

### 常用命令

#### wsl -l -v

查看当前已安装的Linux发行版及对应的wsl版本号

#### wsl --shutdown

关闭所有正在运行的Linux发行版，但是可能出现关不上的情况

### 存储迁移

WSL默认安装的发型本存储在C盘，随着使用的增多，会占用较多空间，建议提早移出到别的盘符

#### 查看运行状态

打开终端控制台，输入`wsl -l -v`查看WSL虚拟机的状态

``` shell
  NAME            STATE           VERSION
* Ubuntu-20.04    Running         2
```

#### 关闭正在运行的虚拟机

使用命令`wsl --shutdown`，再次查看WSL虚拟机的状态，可发现已经关闭

``` shell
  NAME            STATE           VERSION
* Ubuntu-20.04    Stopped         2
```

#### 导出备份

首先在自己选择想要存放wsl的盘符（这里选择D盘）下创建文件夹（名称为WSL）。注意命令中export后是自己Ubuntu的版本名称。

``` shell
wsl --export Ubuntu-20.04 D:\WSL\Ubuntu.tar
```

#### 注销原有版本

``` shell
wsl --unregister Ubuntu-20.04
```

#### 将备份文件恢复到自己新建的文件夹（这里为WSL）下

``` shell
wsl --import Ubuntu-20.04 D:\WSL D:\WSL\Ubuntu.tar
```

#### 恢复默认用户

导入成功后，打开虚拟机默认当前用户会变成root，如果想还原到之前的用户，可执行下述命令，将用户名替换成默认进入虚拟机的用户名即可

``` shell
Ubuntu2004 config -default-user 用户名
```

{% note warning %} 

执行配置默认用户命令时，控制台可能会报错，如果百度后无法解决的情况下， 可以root身份先进入虚拟机，执行`sudo vim /etc/wsl.conf`创建或修改文件，在其中添加如下内容

``` shell
[user]
default = root
```

保存配置并退出，同样在关闭 wsl 之后重新进入，便会发现默认用户已经修改了。

需要注意的是 wsl.conf 配置优先级要高于Ubuntu2004.exe config --default-user，因此如果两个都配置了的话，会以 wsl.conf 中的配置优先。

{% endnote %}
