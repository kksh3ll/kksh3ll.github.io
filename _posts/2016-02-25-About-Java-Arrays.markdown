---
layout: post
title:  关于Java中的数组
date :  2016-02-25 20:34:25
categories: [Java]
---
## 什么是数组

数字是一个简单的复合数据类型，是一系列有序数据的集合，其中的每一个数据都具有相同的数据类型，通过数组名加一个不会越界的下标值来唯一确定数组中的元素。数组是一个特殊的对象。

{% highlight Java %}
public class Test {  
    public static void main(String[] args) {  
        int[] array = new int[10];  
        System.out.println("array的父类是：" + array.getClass().getSuperclass());  
        System.out.println("array的类名是：" + array.getClass().getName());  
    }  
}  
#=> array的父类是：class java.lang.Object
#=> array的类名是：[I  表示一个一维数组
{% endhighlight %}

可以看到数组类和普通类是不一样的，是一种特殊的类。

## 数组的使用方法

数组的使用分四个步骤：
1. 声明数组，数组的类型是int[] array还是int array[]。
2. 分配空间，计算机需要分配多少连续的空间。
3. 赋值，在以分配的空间里放入数据。
4. 处理，对数组的元素进行操作，通过数组名和下标。
