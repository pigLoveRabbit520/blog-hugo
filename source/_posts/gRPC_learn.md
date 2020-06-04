title: gRPC入门使用
author: Salamander
tags:
  - rpc
  - go
categories:
  - Go
  - gRPC
date: 2020-06-02 09:00:00
---
## gRPC是什么
[官网]((https://grpc.io/))如此写到
> A high-performance, open source universal RPC framework

其实，gRPC是一个高性能的，通用的，面向服务端和移动端，基于 HTTP/2 设计的RPC框架。

## RPC框架是什么？
RPC 框架说白了就是让你可以像调用本地方法一样调用远程服务提供的方法，而不需要关心底层的通信细节。简单地说就让远程服务调用更加简单、透明。
RPC框架包含了客户端（Client）和服务端（Server）  
常见的RPC框架有
* gRPC。谷歌出品
* Thrift。Apache出品
* Dubbo。阿里出品，也是一个微服务框架

## gRPC的特性
看[官方文档](https://grpc.io/)的介绍，有以下4点特性：
1. 使用Protocal Buffers这个强大的结构数据序列化工具
2. gRPC可以跨语言使用
3. 安装简单，扩展方便（用该框架每秒可达到百万个RPC）
4. 基于HTTP2协议

## gRPC使用流程
* 定义标准的proto文件
* 生成标准代码（用`protoc`工具）
* 服务端使用生成的代码提供服务
* 客户端使用生成的代码调用服务


## Golang实践

### 安装protoc
首先，我们需要安装`protoc`，这个工具是`Protocol Buffer`的编译器，把proto文件翻译成不同语言（Java，Go等）。  
地址：[protobuf/releases](https://github.com/protocolbuffers/protobuf/releases)  
解压把bin目录下**protoc**文件放到/usr/local/bin目录下即可。

### 安装 protoc-gen-go
`protoc-gen-go`是Go的protoc编译插件，protobuf内置了许多高级语言的编译器，但没有Go的。
```
$ protoc -h
...
--cpp_out=OUT_DIR           Generate C++ header and source.
  --csharp_out=OUT_DIR        Generate C# source file.
  --java_out=OUT_DIR          Generate Java source file.
  --js_out=OUT_DIR            Generate JavaScript source.
  --objc_out=OUT_DIR          Generate Objective C header and source.
  --php_out=OUT_DIR           Generate PHP source file.
  --python_out=OUT_DIR        Generate Python source file.
  --ruby_out=OUT_DIR          Generate Ruby source file.
...
```
```
go get -u github.com/golang/protobuf/protoc-gen-go
```
因为墙的原因，上面的go get命名会失败，我们需要如下设置代理：
```
export http_proxy=http://127.0.0.1:10809
export https_proxy=http://127.0.0.1:10809

# git设置代理
git config --global http.https://go.googlesource.com.proxy socks5://127.0.0.1:10808
git config --global http.https://go.googlesource.com.proxy socks5://127.0.0.1:10808
```









参考：
* [gRPC详细入门教程，Golang/Python/PHP多语言讲解](https://www.cnblogs.com/chenqionghe/p/12394845.html)