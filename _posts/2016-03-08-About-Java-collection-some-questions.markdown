---
layout: post
title:  "关于Java集合的几个常见问题"
date:   2016-03-08 20:26:05
categories: [Java]
---

列出几个集合常见的问题

* `ArrayList`是使用数组实现的list，可以通过索引获取元素。如果数组已满的话需要通过分配新数组将原来的元素移动过去。添加和删除元素需要移动其他元素。`LinkedList`则是链表结构，获取某个元素需要遍历list，但添加和删除元素很快。
 即如果对list增加和删除较多，优先使用`LinkedList`；如果查询较多，则推荐使用`ArrayList`。

* 对list的复制有俩种方法，一种是使用`ArrayList`构造器,另一种使用`Collections.copy()`。
  {% highlight Java %}
   ArrayList<String> newList = new  ArrayList<String>(list);
  {% endhighlight %}

*  删除`ArrayList`中的重复元素，可以将list放入set中来删除 重复元素，然后再放回list中。
  {% highlight Java %}
  Set<String> set = new HashSet<String>(list);
  list.clear();
  list.addAll(set);
  {% endhighlight %}
