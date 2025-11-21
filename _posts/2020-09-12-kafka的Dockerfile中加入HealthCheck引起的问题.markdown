---
layout: post
title: "kafka的Dockerfile中加入HealthCheck引起的问题"
date: 2020-09-12
categories: [Linux]
---

> 

## 起因

kafka服务在运维平台上每次启动都是正在运行，但其实内部服务不一定可用，于是打算加入docker的健康检查机制
`HEALTHCHECK --interval=5s --timeout=2s --retries=12000 CMD curl --silent --fail 127.0.0.1:9092 || exit 1`

加入了以后启动的时候发现报了一个很奇怪的错误：
`org.apache.kafka.common.network.InvalidReceiveException: Invalid receive (size = 1126585245  larger than 104857600)`

网上找的大部分解决办法是配置`socket.request.max.bytes`参数


具体原因看到一篇文章，说是

KAFKA-3990: 当Kafka服务端接受到一个HTTP服务的时候，生成的数据会特别大


[详细][kafka]
[kafka]:    https://www.codenong.com/cs109820568/