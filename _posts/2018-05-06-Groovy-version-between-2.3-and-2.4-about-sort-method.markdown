---
layout: post
title: Groovy版本2.3和2.4之间sort方法的一些差异
date: 2018-05-06
categories: [Groovy]
---

## 起因

编译IDEA的时候，发现了一个Groovy版本冲突的错误，为了兼容性，删掉了高版本的Groovy2.4.12，保留了2.3.0版本，这次没有报告错误，但却引发了另外的更严重的错误，许多的类型不匹配错误，代码肯定是没有什么问题的，初步怀疑还是版本的问题，因为删除了高版本保留了低版本，对使用高版本的一些特性或者方法会出现不兼容。

![img](/img/IDEA20180506.jpg)

## 解决

查找了Groovy的相关文档，文档比较简略，寥寥数语没看出有什么差异的地方，这个怀疑是不是JDK导致的问题，因为在这个项目中有jdk6的API，而项目引用的库是1.6但需要的是1.8，百度了好久依然无果，左思右想，对Groovy也不是很熟，干脆换了Google搜索sorted方法，结果第一条就是答案，踏破铁鞋无觅处，得来全不费功夫啊。

![img](/img/Google20180506.jpg)

谷歌搜索就是好用啊，许多大神都用Google都是有道理的。

轻松找到了问题了所在: ([`引用链接`][引用链接])

- Since Groovy 2.4 we have two new methods which by default return a new collection: toSorted and toUnique.

![img](/img/sorted20180506.jpg){:width="800"}

 [引用链接]: http://mrhaki.blogspot.com/2015/03/groovy-goodness-new-methods-to-sort-and.html
