---
layout: post
title:  HashMap和HashTable的区别
date :  2015-04-15 19:48:12
categories: Java
---

## HashMap 不是线程安全的

`HashMap`是Map的一个子接口，是将键映射到值的对象，其中键和值都是对象，并且不能包含重复键，但可以包含重复值。`HashMap`允许`nullkey`和`null value`。

## `HashTable`是线程安全的一个`Collection`

`HashMap`是`HashTable`的轻量级实现（非线程安全的实现），他们都完成了Map的接口，主要的区别在于`HashMap`允许空键值，由于非线程安全，效率上可能高于`HashTable`。`HashMap`把`HashTable`的`contains`方法去掉了，改成`containsValue`和`containsKey`，因为`contains`方法容易引起误解，`HashTable`继承自`Dictionary`类。