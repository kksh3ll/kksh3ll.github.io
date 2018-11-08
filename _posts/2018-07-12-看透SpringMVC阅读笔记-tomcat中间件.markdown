---
layout: post
title: 看透SpringMVC阅读笔记-tomcat中间件
data: 2018-06-26
categories: Java
---

## Tomcat的顶层结构

Tomcat的顶层容器叫Server，代表整个服务器。Server至少包含一个service，service包含Connector和Containner。

- Connector处理连接相关的事情，提供socket和request、response的转换，可以有多个

- Containner用于封装管理Servlet，具体处理request的请求，只能有一个

![img](/img/tomcat20180626.jpeg)

org.apache.catalina.startup.Catalina类是整个tomcat的管理类，有load、start、stop分别管理整个服务器的生命周期，load方法根据conf/server.xml文件创建Server并调用Server的init方法初始化。

Catalina中的await方法直接调用Server的await方法进入循环，让主线程不会退出。

Tomcat的入口main方法在org.apache.catalina.startup.Bootstrap中，Bootstrap的作用类似一个CatalinaAdaptor，具体处理过程还是由Catalina来完成，这样做的好处是把启动的入口和具体的管理类分开，从而更方便的创建出多种启动方式，每种启动方式只需要写一个相应的CatalinaAdaptor就可以。

## Containner分析

Containner一共有4个子接口Engine、Host、Context、Wrapper和一个默认实现类ContainnerBase，每个子接口都是一个容器，这4个字容器有一个对应的StandardXXX实现类，并且这些实现类都继承ContainnerBase类。通常使用的Servlet封装在Wrapper中。

![img](/img/containner20180626.jpg){:width="800"}

- Engine管理多个Host，一个Service只能有一个Engine

- 每个Host代表一个虚拟主机

- 每个Context代表一个应用

- 每个Wrapper封装着一个Servlet

