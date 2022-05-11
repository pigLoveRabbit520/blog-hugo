title: ASP.NET in Windows Container
author: pigLoveRabbit
tags:
  - Docker
  - WIndows
categories:
  - Docker
date: 2022-05-11 10:19:00
---
平常我们用的都是`Linux Container`，这些容器用的都是Linux的内核，而今天我们要记录的是`Windows Container`，就是讲这些容器用的是Windows的内核，Windows内核是啥？那就是[Windows NT](https://zh.m.wikipedia.org/zh-hans/Windows_NT)。  
查看你的Windows内核版本，可以用
```PowerShell
Get-ComputerInfo | Select WindowsProductName, WindowsVersion, WindowsInstallationType, OsServerLevel, OsVersion, OsHardwareAbstractionLayer
```
类似输出
```
WindowsProductName         : Windows 10 Pro for Workstations
WindowsVersion             : 2009
WindowsInstallationType    : Client
OsServerLevel              :
OsVersion                  : 10.0.19044
OsHardwareAbstractionLayer : 10.0.19041.1566
```
上面NT的版本是10.0.19044。

环境  
* 安装 [.NET 6 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/6.0)
* Windows Server 2022
* Docker（针对Windows Server的docker，没有UI的）
* 安装一个喜欢的代码编辑器，例如 Visual Studio(Code)。

<!-- more -->

## Docker
若要在 Windows Server 上安装 Docker，可以使用由 Microsoft 发布的 OneGet 提供程序 PowerShell 模块（称为 DockerMicrosoftProvider）。 此提供程序启用 Windows 中的容器功能，并安装 Docker 引擎和客户端。 以下是操作方法：  

打开提升的 PowerShell 会话，从 PowerShell 库安装 Docker-Microsoft PackageManagement 提供程序。

```PowerShell
Install-Module -Name DockerMsftProvider -Repository PSGallery -Force
```
如果系统提示安装 NuGet 提供程序，还请键入 Y 进行安装。  

如果在打开 PowerShell 库时遇到错误，则可能需要将 PowerShell 客户端使用的 TLS 版本设置为 TLS 1.2。 为此，请运行以下命令：

```PowerShell
# Set the TLS version used by the PowerShell client to TLS 1.2.
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12;
```
使用 PackageManagement PowerShell 模块安装最新版本的 Docker。  

```PowerShell
Install-Package -Name docker -ProviderName DockerMsftProvider
```
PowerShell 询问是否信任包源“DockerDefault”时，键入 A 以继续进行安装。  

在安装完成后，请重启计算机。

```PowerShell
Restart-Computer -Force
```

## Windows Container
我们知道Linux容器的Base Image是alpine或者scratch， 那Windows容器的Base Image是什么呢？其实微软官方也有[介绍](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-base-images)了。  

![upload successful](/images/windows_base_image.png)  
好了，我们进入正题。  

要确保当前Docker是使用`Windows Container`的，我们可以拉个镜像
```
docker pull mcr.microsoft.com/windows/nanoserver:ltsc2022-amd64
```
如果看到错误消息“no matching manifest for linux/amd64 in the manifest list entries”，那说明 Docker 用的是Linux 容器。   
注意：`mcr.microsoft.com`上的镜像在国内访问挺慢的，你可以先把镜像pull到阿里云，然后再在你电脑上拉取。  
我们跑一个简单容器（执行cmd）
```
docker run -it mcr.microsoft.com/windows/nanoserver:ltsc2022-amd64  cmd.exe
```
在容器中执行一些命令：
```
C:\>dir
 Volume in drive C has no label.
 Volume Serial Number is F63B-D098

 Directory of C:\

05/05/2022  10:35 AM             5,510 License.txt
05/05/2022  10:37 AM    <DIR>          Users
05/11/2022  03:37 PM    <DIR>          Windows
               1 File(s)          5,510 bytes
               2 Dir(s)  21,302,714,368 bytes free
               
C:\>hostname
fbb1d7595c03

C:\>ipconfig/all

Windows IP Configuration

   Host Name . . . . . . . . . . . . : fbb1d7595c03
   Primary Dns Suffix  . . . . . . . :
   Node Type . . . . . . . . . . . . : Hybrid
   IP Routing Enabled. . . . . . . . : No
   WINS Proxy Enabled. . . . . . . . : No
   DNS Suffix Search List. . . . . . : lan

Ethernet adapter vEthernet (Ethernet):

   Connection-specific DNS Suffix  . : lan
   Description . . . . . . . . . . . : Hyper-V Virtual Ethernet Container Adapter
   Physical Address. . . . . . . . . : 00-15-5D-E5-CC-89
   DHCP Enabled. . . . . . . . . . . : No
   Autoconfiguration Enabled . . . . : Yes
   Link-local IPv6 Address . . . . . : fe80::a0c8:ad95:72dd:fe6f%17(Preferred)
   IPv4 Address. . . . . . . . . . . : 172.30.150.168(Preferred)
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . : 172.30.144.1
   DNS Servers . . . . . . . . . . . : 172.30.144.1
                                       10.0.2.3
   NetBIOS over Tcpip. . . . . . . . : Disabled
   Connection-specific DNS Suffix Search List :
                                       lan
```
可以看到，容器中目录还有hostname，网络都是隔离的。  


## 运行一个ASP.NET程序
dotnet 6已经可以很方便的创建asp程序的骨架了，创建新目录myweb, 终端输入`dotnet new web`，然后sdk就帮你创建了一个hello world程序。  
看一下Program.cs的内容：
```
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.Run();
```
恩，蛮简单的，我们在宿主机上运行`dotnet run`，输出
```
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:7174
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5017
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /your_path/myweb/
```
`curl http://localhost:5017`返回了`Hello World!`，速度阔以的。  


为了在Docker中运行程序，我们需要创建一个Dockerfile
```
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
WORKDIR /app

# Copy everything
COPY . ./
# Restore as distinct layers
RUN dotnet restore
# Build and publish a release
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "myweb.dll"]
```




参考：
* [入门：准备适用于容器的 Windows](https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/quick-start/set-up-environment?tabs=Windows-Server)
* [Tutorial: Containerize a .NET app](https://docs.microsoft.com/en-us/dotnet/core/docker/build-container?tabs=windows)