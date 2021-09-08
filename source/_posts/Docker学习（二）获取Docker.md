---
title: Docker学习（二）获取Docker
date: 2021-09-08 17:04:41
categories: 
    - Docker
cover: /img/post_cover/Docker_02.jpg
tags:
    - Docker
---
# Docker学习（二）获取Docker

您可以在多个平台(Mac/Windows/Linux)上[下载](https://docs.docker.com/get-docker/)并安装 Docker。
## 1. 在Windows上安装Docker Desktop前置条件
对于仍在微软服务时间表内的 Windows 10版本，Docker 只支持 Windows 上的 Docker 桌面。
### 1.1 系统需求
#### WSL(Windows Subsystem for Linux) 2后端  
- Windows 10 64bit: 家庭版或者专业版2004（build 19041）或者更高版本 
   Windows 10 64bit: 企业版或者教育版1909（build 18363） 或者更高版本
- 在Windows上启用WSL2特性，更多细节请参考[Miscrosoft documentation](https://docs.microsoft.com/en-us/windows/wsl/install-win10).
- 在Windows 10上成功运行WSL 2需要一下先决条件：
64位处理器和Second Level Address Translation(SLAT)（二级地址转换）
4GB系统内存
必须在BIOS设置中启用BIOS级虚拟化支持。更多信息参考[Virtualization](https://docs.docker.com/docker-for-windows/troubleshoot/#virtualization-must-be-enabled).
- 下载并安装Linux内核更新包。[在 Windows 10 上安装 WSL | Microsoft Docs](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10#step-4---download-the-linux-kernel-update-package)
#### Hyper-V 后端和Windows容器
- Windows 10 64-bit: 专业版2004 (build 19041) 或者更高版本 
    Windows 10 64bit: 企业版或者教育版1909（build 18363） 或者更高版本
- 必须启用 Hyper-V 和 Containers Windows 特性
- 在Windows 10上成功运行WSL 2需要一下先决条件：
 - 64位处理器和Second Level Address Translation(SLAT)（二级地址转换）
 - 4GB系统内存
 - 必须在BIOS设置中启用BIOS级虚拟化支持。更多信息参考[Virtualization](https://docs.docker.com/docker-for-windows/troubleshoot/#virtualization-must-be-enabled).
#### Docker Desktop Installer包含哪些内容？
Docker Desktop包含[Docker Engine](https://docs.docker.com/engine/)，Docker CLI client，[Docker Compose](https://docs.docker.com/compose/)，Docker Content Trust，[Kubernetes](https://github.com/kubernetes/kubernetes/)和[Credential Helper](https://github.com/docker/docker-credential-helpers/)。  
使用 Docker Desktop 创建的容器和镜像在安装它的计算机上的所有用户帐户之间共享。这是因为所有 Windows 帐户都使用相同的 VM 来构建和运行容器。请注意，在使用 Docker Desktop WSL 2后端时，不可能在用户帐户之间共享容器和镜像。  
嵌套的虚拟化场景，比如在 VMWare 或 Parallels 实例上运行 Docker Desktop 可能可以使用，但这并不能保证。有关的更多信息，请参考[在嵌套虚拟化场景中运行 Docker 桌面](https://docs.docker.com/docker-for-windows/troubleshoot/#running-docker-desktop-for-windows-in-nested-virtualization-scenarios)。
#### 关于Windows容器
- [在Windows容器之间切换](https://docs.docker.com/docker-for-windows/#switch-between-windows-and-linux-containers)
- [开始使用windows容器（Lab）](https://github.com/docker/labs/blob/master/windows/windows-containers/README.md)
- [适用于Windows的Docker容器平台的文章和博客](https://docs.docker.com/docker-for-windows/troubleshoot/#limitations-of-windows-containers-for-localhost-and-published-ports)

### 1.2 Install Docker Desktop on Windows
#### 查看自己的电脑系统是否满足系统需求
开始-设置-系统-关于，如下示例：
**设备规格**
```
设备名称	ESR-20151030WHY
处理器	Intel(R) Core(TM) i5-4200U CPU @ 1.60GHz   2.30 GHz
机带 RAM	8.00 GB
设备 ID	9E4CA172-84E3-48EB-8909-8E0CCD2D66A1
产品 ID	00330-50000-00000-AAOEM
系统类型	64 位操作系统, 基于 x64 的处理器
笔和触控	没有可用于此显示器的笔或触控输入
```
**Windows规格**
```
版本	Windows 10 专业版
版本号	21H1
安装日期	‎2021/‎7/‎12
操作系统内部版本	19043.1083
体验	Windows Feature Experience Pack 120.2212.3530.0
```
**[Enable the Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10)**
1. 以管理员身份打开 PowerShell 并运行:
```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

```
结果如下：
```
Windows PowerShell
版权所有 (C) Microsoft Corporation。保留所有权利。

尝试新的跨平台 PowerShell https://aka.ms/pscore6

PS C:\WINDOWS\system32> dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

部署映像服务和管理工具
版本: 10.0.19041.844

映像版本: 10.0.19043.1083

启用一个或多个功能
[==========================100.0%==========================]
操作成功完成。
PS C:\WINDOWS\system32>

```

![docker install](docker_02_01.png)

2. Check requirements for running WSL 2
要更新到 wsl2，您必须运行 Windows 10.
- For x64 systems: Version 1903 or higher, with Build 18362 or higher.
- For ARM64 systems: Version 2004 or higher, with Build 19041 or higher.
- Builds lower than 18362 do not support WSL 2. Use the [Windows Update Assistant](https://www.microsoft.com/software-download/windows10) to update your version of Windows.
To check your version and build number, select Windows logo key + R, type winver, select OK. [Update to the latest Windows version](http://ms-settings:windowsupdate/) in the Settings menu.
3. Enable Virtual Machine feature
在安装 wsl2之前，您必须启用虚拟机平台的可选特性。您的计算机将需要虚拟化能力来使用这个特性。
以管理员身份打开 PowerShell 并运行:
```
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```
重新启动计算机以完成 WSL 安装并更新到 wsl2。
4. Download the Linux kernel update package 
下载最新软件包:
- [WSL2 Linux kernel update package for x64 machines 针对 x64机器的 WSL2 Linux 内核更新包](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)
- 运行上一步中下载的更新包。(双击运行——系统将提示您提高权限，选择“是”以批准此安装。)
5. 将 WSL 2 设置为默认版本
打开 PowerShell 并运行以下命令，在安装新的 Linux 发行版时将 WSL 2设置为默认版本:
```
Windows PowerShell
版权所有 (C) Microsoft Corporation。保留所有权利。

尝试新的跨平台 PowerShell https://aka.ms/pscore6

PS C:\WINDOWS\system32> wsl --set-default-version 2
有关与 WSL 2 的主要区别的信息，请访问 https://aka.ms/wsl2
PS C:\WINDOWS\system32>

```
**检查系统的处理器，系统RAM，BIOS中是否开启了虚拟化**
打开 PowerShell 并运行以下命令
```
Windows PowerShell
版权所有 (C) Microsoft Corporation。保留所有权利。

尝试新的跨平台 PowerShell https://aka.ms/pscore6

PS C:\WINDOWS\system32> systeminfo

主机名:           ESR-20151030WHY
OS 名称:          Microsoft Windows 10 专业版
OS 版本:          10.0.19043 暂缺 Build 19043
OS 制造商:        Microsoft Corporation
OS 配置:          独立工作站
OS 构建类型:      Multiprocessor Free
注册的所有人:     微软用户
注册的组织:       微软中国
产品 ID:          00330-50000-00000-AAOEM
初始安装日期:     2021/7/12, 21:17:33
系统启动时间:     2021/8/1, 16:05:23
系统制造商:       LENOVO
系统型号:         80EQCTO1WW
系统类型:         x64-based PC
处理器:           安装了 1 个处理器。
                  [01]: Intel64 Family 6 Model 69 Stepping 1 GenuineIntel ~1600 Mhz
BIOS 版本:        LENOVO 9DCN26WW(V2.07), 2014/9/23
Windows 目录:     C:\WINDOWS
系统目录:         C:\WINDOWS\system32
启动设备:         \Device\HarddiskVolume1
系统区域设置:     zh-cn;中文(中国)
输入法区域设置:   zh-cn;中文(中国)
时区:             (UTC+08:00) 北京，重庆，香港特别行政区，乌鲁木齐
物理内存总量:     8,120 MB
可用的物理内存:   4,186 MB
虚拟内存: 最大值: 16,312 MB
虚拟内存: 可用:   11,908 MB
虚拟内存: 使用中: 4,404 MB
页面文件位置:     C:\pagefile.sys
域:               WorkGroup
登录服务器:       \\ESR-20151030WHY
修补程序:         安装了 4 个修补程序。
                  [01]: KB5003254
                  [02]: KB5000736
                  [03]: KB5004945
                  [04]: KB5003742
网卡:             安装了 2 个 NIC。
                  [01]: Realtek PCIe GBE Family Controller
                      连接名:      本地连接
                      状态:        媒体连接已中断
                  [02]: Qualcomm Atheros AR956x Wireless Network Adapter
                      连接名:      无线网络连接
                      启用 DHCP:   是
                      DHCP 服务器: 192.168.0.1
                      IP 地址
                        [01]: 192.168.0.195
                        [02]: fe80::78dd:61d6:a6e8:e5d4
Hyper-V 要求:     已检测到虚拟机监控程序。将不显示 Hyper-V 所需的功能。
PS C:\WINDOWS\system32>
```
![cpu](docker_02_02.png)

## 2. 在 Windows 上安装 Docker 桌面
- 双击 Docker Desktop Installer.exe 运行安装程序。
- 当提示时，确保启用 Hyper-V Windows 特性或者安装 WSL 2所需的 Windows 组件选项在 Configuration 页面上被选中。
![docker configuration](docker_02_03.png)
![docker configuration](docker_02_04.png)
![docker configuration](docker_02_05.png)
- 按照安装向导上的说明授权安装程序并继续安装。
![docker configuration](docker_02_06.png)
- 安装成功后，单击“关闭”完成安装过程。
![doccker configuration](docker_02_07.png)
- 如果您的管理帐户与您的用户帐户不同，则必须将该用户添加到 docker-users 组中。以管理员身份运行计算机管理并导航到 `Local Users and Groups > Groups > docker-users`。右击可将用户添加到组中。注销并重新登录以使更改生效。
## 3. 启动Docker桌面
点击Docker Desktop图标即可启动。
![docker configuration](docker_02_08.png)
![docker configuration](docker_02_09.png)
当状态栏中的鲸图标保持稳定时，Docker 桌面就会启动并运行，并且可以从任何终端窗口访问。要了解更多信息，请参见[Docker设置](https://docs.docker.com/docker-for-windows/#docker-settings-dialog)。
## 4. 快速入门指南
要根据需要运行快速启动指南，右键单击通知区域(或系统托盘)中的 Docker 图标，以打开 Docker Desktop 菜单，然后选择快速启动指南。

## 5. 更新
从 Docker Desktop 3.0.0开始，Docker Desktop 的更新将作为以前版本的 delta 更新提供。当更新可用时，dockerdesktop 会显示一个图标，指示更新版本的可用性。您可以选择何时开始下载和安装过程。
![docker configuration](docker_02_10.png)
单击“下载更新”当您准备好下载更新时。这将在后台下载更新。下载更新后，单击 Update 并从 Docker 菜单重新启动。这将安装最新更新并重新启动 Docker Desktop 以使更改生效。
## 6. 卸载Docker Desktop
从Windows卸载Docker Desktop:
- 点击Windows的开始菜单，选择设置-应用-应用和功能
- 在应用列表中选择Docker Desktop，选择卸载
- 点击卸载确认你的选择。