---
layout: post
title:  Java中Iterator的用法
date:   2015-04-12
categories: [Java]
---

## 迭代器

由于Java中数据容器众多，而对数据容器的操作在很多时候都具有极大的共性，于是Java采用了迭代器为各种容器提供公共的操作接口。

使用Java的迭代器iterator可以使得对容器的遍历操作完全与其底层相隔离，可以到达极好的解耦效果。

迭代器是一种设计模式，它是一个对象，它可以遍历并选择序列中的对象，Java中的Iterator功能比较简单，并且只能单向移动：

* 使用方法iterator()要求容器返回一个Iterator。第一次调用Iterator的next()方法时，返回序列的第一个元素。
* 使用next()获得序列中的下一个元素。
* 使用hasNext()检查序列中是否还有元素。
* 使用remove()将迭代器新返回的元素删除。

{% highlight Java %}
List l = new ArrayList();
l.add("aa");
l.add("bb");
l.add("cc");
for(Iterator iter = l.iterator();iter.hasNext();){
	String s = (String)iter.next();
	System.out.println(str);
}

{% endhighlight %}





























