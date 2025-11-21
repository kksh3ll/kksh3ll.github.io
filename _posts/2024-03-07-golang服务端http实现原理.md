---
layout: post
title: "golang服务端http实现原理"
date: 2024-03-07
categories: [golang]
---

> 

## 服务端 Server

![img](/img/202403071.webp)

### 主要有这些流程：

- 注册handler到map中，map的key是键值路由

- handler注册完之后就开启循环监听，监听到一个连接就会异步创建一个 Goroutine

- 在创建好的 Goroutine 内部会循环的等待接收请求数据

- 接受到请求后，根据请求的地址去处理器路由表map中匹配对应的handler，然后执行handler

## 监听和服务启动

![img](/img/202403072.webp)

net.Listen 实现了TCP协议上监听本地的端口8080 (ListenAndServe()中传过来的)，Server.Serve接受 net.Listener实例传入，然后为每个连接创建一个新的服务goroutine

使用net.Listen函数实现网络监听需要经过以下几个步骤：

- 调用net.Listen函数，指定网络类型和监听地址。

- 使用listener.Accept函数接受客户端的连接请求。

- 在一个独立的goroutine中处理每个连接。

- 在处理完连接后，调用conn.Close()来关闭连接

参考链接：[图文吃透Golang net/http 标准库][server]
[server]: https://mp.weixin.qq.com/s/e7Z_kZrayTFx7y0hlzoTdg