---
layout: post
title: 看透SpringMVC阅读笔记
date: 2018-06-26
categories: [Java]
---

## NioSocket的用法

NioSocket中服务端的处理可以分为5步：

- 创建`ServerSocketChannel`并设置相应的参数

- 创建`Selector`并注册到ServerSocketChannel上

- 调用Selector的select方法等待请求

- Selector接受到请求后使用`selectedKeys`返回`SelectionKey`集合

- 使用SelectionKey获取到Channel、Selector和操作类型并进行具体操作