---
layout: post
title: 看透SpringMVC阅读笔记
data: 2018-06-26
categories: Java
---

## NioSocket的用法

NioSocket中服务端的处理可以分为5步：

- 创建ServerSocketChannel并设置相应的参数

- 创建Selector并注册到ServerSocketChannel上

- 调用Selector的select方法等待请求

- Selector接受到请求后使用selectedKeys返回SelectionKey集合

- 使用SelectionKey获取到Channel、Selector和操作类型并进行具体操作