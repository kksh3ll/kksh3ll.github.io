---
layout: post
title: 【Array/String】leetcode算法题之一
date: 2017-06-03
categories: [Java]
---

## 问题

给出一个整数数组，找出俩个数，这俩个数加起来之和是给定的目标值。

## 解决方案

1. 暴力破解法：

	* 时间复杂度：O(n<sup>2</sup>)
	* 空间复杂度：O(1)

2. Hash table

	* 时间复杂度：O(n)
	* 空间复杂度：O(n)

Java代码如下：

{% highlight Java %}
public int[]twoSum(int[] numbers, int target) {
	Map<Integer, Integer> map = new HashMap<>();
	for (int i = 0; i < numbers.length; i++) {
		int x = numbers[i];
		if (map.containsKey(target - x)) {
			return new int[] { map.get(target - x), i }
		}
		map.put(x, i);
	}
	throw new IllegalArgumentException("No two sum solution!");
}
{% endhighlight %}

# 疑问

1. 高手的解答真是让菜鸟我捉摸不透，返回的数组怎么保证前一比后一小呢？

2. 万一俩个数是相等的，Hash table貌似不太适合
