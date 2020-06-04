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

