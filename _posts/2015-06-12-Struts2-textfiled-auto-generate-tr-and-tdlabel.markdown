---
layout: post
title : s:textfield等struts-tags去掉自动生成的tr和td标签
date  : 2015-06-12 23:15:45
categories: Java
---

* 在使用struts2的时候发现如果按照默认的方式使用<s:textfield>标签，会自动加上<tr><td>标签，比如

`<s:textfield key="username"/>`

会显示成

`<tr><td><input type=text name=username/></td></tr> `

* 有时候并不需要这些td tr，所以可以这样写

`<s:textfield key="username" theme="simple"/>` 

* 或者在struts.xml中加入

`<constant name="struts.ui.theme" value="simple" />`

这样全部默认使用simple

注:默认的theme是xhtml
