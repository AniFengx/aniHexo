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

### 网络问题

使用 wsl2 版本，在宿主机和 wsl 中使用 localhost 都可访问到部署的服务

> 使用远程 IP 地址连接到应用程序时，它们将被视为来自局域网 (LAN) 的连接。 这意味着你需要确保你的应用程序可以接受 LAN 连接。
>
> 例如，你可能需要将应用程序绑定到 0.0.0.0 而非 127.0.0.1。 以使用 Flask 的 Python 应用为例，可以通过以下命令执行此操作：app.run(host='0.0.0.0')。 进行这些更改时请注意安全性，因为这将允许来自你的 LAN 的连接。

#### 从局域网 (LAN) 访问 WSL 2 分发版

WSL 2 有一个带有其自己独一无二的 IP 地址的虚拟化以太网适配器。目前，若要启用此工作流，你需要执行与常规虚拟机相同的步骤。

下面是使用 [Netsh 接口 portproxy](https://learn.microsoft.com/zh-cn/windows-server/networking/technologies/netsh/netsh-interface-portproxy1 "Netsh 接口 portproxy")  Windows 命令添加端口代理的示例，该代理侦听主机端口并将该端口代理连接到 WSL 2 VM 的 IP 地址。

``` shell
netsh interface portproxy add v4tov4 listenport=<yourPortToForward> listenaddress=0.0.0.0 connectport=<yourPortToConnectToInWSL> connectaddress=(wsl hostname -I)
```

在此示例中，需要更新 `<yourPortToForward>` 到端口号，例如 `listenport=4000`。 `listenaddress=0.0.0.0` 表示将接受来自任何 IP 地址的传入请求。 侦听地址指定要侦听的 IPv4 地址，可以更改为以下值：IP 地址、计算机 NetBIOS 名称或计算机 DNS 名称。 如果未指定地址，则默认值为本地计算机。 需要将 `<yourPortToConnectToInWSL>` 值更新为希望 WSL 连接的端口号，例如 `connectport=4000`。 最后，`connectaddress` 值必须是通过 WSL 2 安装的 Linux 分发版的 IP 地址（WSL 2 VM 地址），可通过输入命令：`wsl.exe hostname -I` 找到。

因此，此命令可能如下所示：

``` shell
netsh interface portproxy add v4tov4 listenport=4000 listenaddress=0.0.0.0 connectport=4000 connectaddress=192.168.101.100
```

要获取 IP 地址，请使用：

- `wsl hostname -I` 标识通过 WSL 2 安装的 Linux 分发版 IP 地址（WSL 2 VM 地址）
- `cat /etc/resolv.conf` 表示从 WSL 2 看到的 WINDOWS 计算机的 IP 地址 (WSL 2 VM)

{% note info %} 

在主机名命令中使用小写“i”将生成与使用大写“I”不同的结果。 `wsl hostname -i` 是本地计算机（127.0.1.1 是占位符诊断地址），而 `wsl hostname -I` 会返回其他计算机所看到的本地计算机的 IP 地址，应该用于识别通过 WSL 2 运行的 Linux 发行版的 `connectaddress`。

{% endnote %}

#### netsh interface portproxy 常用命令

Portproxy 服务器从服务器侦听的 IPv4 端口和地址列表中删除 IPv4 地址。

``` shell
netsh interface portproxy delete v4tov4 listenport=<yourPortToForward> listenaddress=0.0.0.0
```

显示所有 portproxy 参数，包括 v4tov4、v4tov6、v6tov4 和 v6tov6 的端口/地址对。

``` shell
netsh interface portproxy show all
```

#### 镜像模式

部分情况下，wsl 中的应用程序需要 wsl 虚拟机所在 ip 和宿主机保持一致，这时可启用 wsl 的镜像模式，[参考](https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config#wslconfig "WSL 中的高级设置配置") 

开启方式为修改 .wslconfig 文件，使用 .wslconfig 为 WSL 上运行的所有已安装的发行版配置全局设置。

- 默认情况下，.wslconfig 文件不存在。 它必须创建并存储在`%UserProfile%`目录中才能应用这些配置设置。
- 用于在作为 WSL 2 版本运行的所有已安装的 Linux 发行版中全局配置设置。
- 只能用于 WSL 2 运行的发行版。 作为 WSL 1 运行的发行版不受此配置的影响，因为它们不作为虚拟机运行。
- 要访问 `%UserProfile%` 目录，请在 PowerShell 中使用 `cd ~` 访问主目录（通常是用户配置文件 `C:\Users\<UserName>`），或者可以打开 Windows 文件资源管理器并在地址栏中输入 `%UserProfile%`。 该目录路径应类似于：`C:\Users\<UserName>\.wslconfig`。

WSL 将检测这些文件是否存在，读取内容，并在每次启动 WSL 时自动应用配置设置。 如果文件缺失或格式错误（标记格式不正确），则 WSL 将继续正常启动，而不应用配置设置。

文件中添加 `[wsl2]` 标签,并在下方添加`networkingMode=mirrored`

``` shell
[wsl2]
networkingMode=mirrored
```

### Windows 与 WSL 文件交互

在 wsl 中可直接访问宿主机，宿主机盘符为`/mnt/`

``` shell
cp -r /mnt/e/BaiduNetdiskDownload/ls16 /home/soap/
```

`/mnt/e`表示本地的e盘，Windows 的文件路径为`/mnt/e/BaiduNetdiskDownload/ls16`，wsl 的目标路径为`/home/soap/`

