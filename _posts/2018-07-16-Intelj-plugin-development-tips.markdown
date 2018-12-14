---
layout: post
title: "Intellij 插件开发注意点"
date: 2018-07-16
categories: [intellij]
---

### 编辑器（Editor）写操作

- 在插件中可能需要向`PsiFile`中写入一些方法或者字段（也可能是写入后需要展示，而不是类似`PsiAugmentProvider`生成的快照），需要注意的是，我们拿到新生成的`psiClass`以后，不能使用`psiClass.add(field)`添加代码，要调用`WriteCommandAction.runWriteCommandAction`写代码，否则会抛出异常：`Must not change PSI outside command or undo-transparent action.`这是因为`Intellij Platform`不允许在主线程中进行实时的文件写入，而需要通过一个异步任务来进行。这时需要调用`WriteCommandAction`来写相关的内容，代码如下：

{% highlight Java %}
WriteCommandAction.runWriteCommandAction(Project, new Runnable() {
      @Override
      public void run() {
        //do something
      }
});
{% endhighlight %}
