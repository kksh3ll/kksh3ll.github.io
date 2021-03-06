---
layout: post
title: Oracle数据库中新建表空间及添加用户
date: 2017-06-03
categories: [Oracle]
---

## 创建表空间

表空间就是dbf数据文件，存储创建表

{% highlight Sql %}
create tablespace kksh3ll
logging
datafile '/u01/app/oracle/product/11.2.0/oradata/kksh3ll/kksh3ll.dbf' --表空间数据文件位置
size 32m
autoextend on
next 32m maxsize 2048m
extent management local;
{% endhighlight %}

## 使用sys创建oracle用户（业务系统连接oracle使用的用户）

创建的用户默认的表空间是`kksh3ll`

{% highlight Sql %}
create user kksh3ll identified by kksh3ll
default tablespace kksh3ll
temporary tablespace temp;
{% endhighlight %}

给用户授权:

`grant connect,resource,dba to kksh3ll;`

退出sys，使用新创建用户登陆。

## 字符集的问题

查看oracle服务端字符集：

`select * from NLS_DATABASE_PARAMETERS; --返回服务端的字符集`

查看oracle客户端字符集：

`select * from V$NLS_PARAMETERS;`
`select userenv('language') from DUAL;`







