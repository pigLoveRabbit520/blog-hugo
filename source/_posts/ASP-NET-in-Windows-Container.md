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

## 设置为Windows Container