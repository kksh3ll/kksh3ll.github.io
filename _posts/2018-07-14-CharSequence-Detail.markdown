---
layout: post
title: String和CharSequence的区别
date: 2018-07-14
categories: [Java]
---

## `String` 和 `CharSequence` 的关系

`String` 继承于`CharSequence`，也就是说`String`也是`CharSequence`类型。

`CharSequence`是一个接口，它只包括`length()`, `charAt(int index)`, `subSequence(int start, int end)`这几个API接口。除了String实现了CharSequence之外，StringBuffer和StringBuilder也实现了CharSequence接口。

需要说明的是，`CharSequence`就是字符序列，`String`, `StringBuilder`和`StringBuffer`本质上都是通过字符数组实现的！

## `StringBuilder` 和 `StringBuffer` 的区别

`StringBuilder` 和 `StringBuffer`都是可变的字符序列。

- 它们都继承于`AbstractStringBuilder`，实现了`CharSequence`接口。

- `StringBuilder`是非线程安全的，而`StringBuffer`是线程安全的。

它们之间的关系图如下：

![img](/img/20180714charseq.jpg)

## `CharSequence` 详解

`CharSequence`就是一个可读的字符序列。对于不同类型的字符序列，这一接口都以统一且只读的方式去读取。一个字符值代表了BMP中的一个字符或者一个代理。(BMP是什么?BMP包含了现代大多数语言的字符集)

这一接口,并没有提炼出`Object`类定义的`equals()`方法和`hashCode()`方法的通用规范(但是像其他的接口，比如Map就有`equals()`方法和
`hashCode()`方法)。因此,使用任意的`CharSequence`实例作为`set`集合的元素或者`map`中的`key`,这种做法都是不合适的(因为`CharSequence`实例是没有`equals`方法和`hashCode`方法的,所以对应的实例的比较就取决于其继承的类或者其他实现的接口,那么两个被比较的类如果因为继承的类或者实现的接口
不同出现什么结果都是不可控的)。

### CharSequence源码

{% highlight Java %}
package java.lang;

public interface CharSequence {

	// 返回字符序列的长度.
    // 长度是16bit的整数倍.(因为String类采用的是UTF-16编码,一个字符占用2个字节长度)
    int length();

    // 返回指定索引index位置处的字符，索引index的范围是[0,length()-1]
    char charAt(int index);
    
    // 返回一个新的字符序列，这个序列是原字符序列的子序列
    // 子序列的截取开始位置为:原序列中start的位置；截取结束位置为:原序列中(end-1)的位置
    // 因此子字符序列的长度为(end-start)，所以如果传入参数start=end,则返回子序列为空序列
    CharSequence subSequence(int start, int end);

	// 返回字符序列的字符串形式，字符串中字符的顺序和字符序列保持一致，字符串的长度和字符序列一致
    public String toString();
}
{% endhighlight %}

