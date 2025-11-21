---
layout: post
title : 使用Mybatis-generator生成底层代码时遇到的问题
date  : 2017-04-11 13:14:29
categories: [Java] 
---

使用springMVC和Mybatis框架开发的时候，实际项目中往往会根据数据库来生成相应的配置文件和底层代码。Mybatis官方推出了自动化工具。

但在使用这个工具的时候遇到了一些问题，其中包括配置文件的问题和在eclipse上使用插件出现的问题

## 官方给出的配置文件 `generatorConfig.xml`

{% highlight javascript %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
  PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
  "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">

<generatorConfiguration>
  <classPathEntry location="/Program Files/IBM/SQLLIB/java/db2java.zip" />

  <context id="DB2Tables" targetRuntime="MyBatis3">
    <jdbcConnection driverClass="COM.ibm.db2.jdbc.app.DB2Driver"
        connectionURL="jdbc:db2:TEST"
        userId="db2admin"
        password="db2admin">
    </jdbcConnection>

    <javaTypeResolver >
      <property name="forceBigDecimals" value="false" />
    </javaTypeResolver>

    <javaModelGenerator targetPackage="test.model" targetProject="\MBGTestProject\src">
      <property name="enableSubPackages" value="true" />
      <property name="trimStrings" value="true" />
    </javaModelGenerator>

    <sqlMapGenerator targetPackage="test.xml"  targetProject="\MBGTestProject\src">
      <property name="enableSubPackages" value="true" />
    </sqlMapGenerator>

    <javaClientGenerator type="XMLMAPPER" targetPackage="test.dao"  targetProject="\MBGTestProject\src">
      <property name="enableSubPackages" value="true" />
    </javaClientGenerator>

    <table schema="DB2ADMIN" tableName="ALLTYPES" domainObjectName="Customer" >
      <property name="useActualColumnNames" value="true"/>
      <generatedKey column="ID" sqlStatement="DB2" identity="true" />
      <columnOverride column="DATE_FIELD" property="startDate" />
      <ignoreColumn column="FRED" />
      <columnOverride column="LONG_VARCHAR_FIELD" jdbcType="VARCHAR" />
    </table>

  </context>
</generatorConfiguration>
{% endhighlight %}

其中`targetProject`的值在使用eclipse插件的时候之间填写项目名，填写相对路径和绝对路径是不会生成代码，但是既不会报错也不会显示相关的日志。我在这个坑上浪费了很长时间。

`classPathEntry`的值是项目连接数据库的驱动，这个在eclipse插件上是没有的，需要自己添加。

在使用这个工具的时候不会覆盖任何的文件，所以也没有什么担心的。

















