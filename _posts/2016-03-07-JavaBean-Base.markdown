---
layout: post
title:  JavaBean的基础知识
date:   2016-03-07
categories: [web]
---
## 简介

* `JavaBean`是可以重复使用的类，没有用户界面，负责业务数据的处理
* 与JSP配合，可以简化JSP代码

## 在JSP中访问`JavaBean`

* 访问`JavaBean`的JSP标签，`jsp:useBean`声明`JavaBean`对象。`import = "org.ComBean"`导入JavaBean类，`id="myBean"`表示引用`JavaBean`对象的局部变量名。`scope="session"`表示特定范围。

{% highlight Java %}
org.ComBean myBean =null;
myBean = session.getAttribute("myBean");
if(myBean==null)
{
    myBean = new org.ComBean();
    session.setAttribute("myBean",myBean);
}
{% endhighlight %}

* 访问JavaBean属性，`<jsp:setProperty name="myBean" property="count" value="10"/>`等效于`myBean.setCount("10")`，`<jsp:getProperty name="myBean" property="count"/>`等效于`myBean.getCount()`。

* `JavaBean`的范围:`scope`属性:
  1. `page`范围，页面范围内，从客户请求访问一个JSP开始到这个JSP文件执行结束
  2. `request`范围，请求范围内，从客户请求访问一个JSP文件开始，到这个JSP文件返回响应结果结束
  3. `session`范围，会话范围内，处于同一个会话范围内的Web组件共享这个会话范围内的JavaBean对象
  4. `application`范围，在Web应用范围内，处于同一个Web应用中的所有Web组件共享这个Web应用范围内的`javaBean`对象




































