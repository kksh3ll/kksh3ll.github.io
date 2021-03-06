---
layout: post
title: 从相关代码认识适配器模式
date: 2018-05-25
categories: [Java]
---

## 适配器模式特点

关于适配器模式就不详细说了，简单描述就是把一个类的接口变成用户所期待的另一种接口，使原本接口不匹配而无法一起工作的类能够在一起工作。

适配器模式在应用的场景中，一是系统需要使用现有的类，而这个类的接口不符合要求，二是想建立一个可重用的类，与一些彼此没有关联的类一起工作，这些类不一定有一致的接口。这个时候可以考虑使用适配器模式来进行开发。

![img](/img/adapter20180525.png)


- 优点：可以让任何两个没有关联的类一起运行、提高了类的复用并且灵活性好。

- 缺点：过多地使用适配器，会让系统非常零乱、由于 JAVA 至多继承一个类，所以至多只能适配一个适配者类，而且目标类必须是抽象类。

## 缺省适配器模式

缺省适配器模式也被称为接口适配器模式，当不需要全部实现接口提供的方法时，可先设计一个抽象类实现接口，并为该接口中每个方法提供一个默认实现（空方法），那么该抽象类的子类可有选择地覆盖父类的某些方法来实现需求，它适用于一个接口不想使用其所有的方法的情况。

IDEA中源码许多部分都使用了这种结构，对其中的部分源码进行解析来学习这种模式

类图如下：

![img](/img/LookupAdapter180902.jpg){:width="800"}

LookupAdapter抽象类实现LookupListener接口，也就是LookupListener的接口适配器，空实现了接口的所有方法。

{% highlight Java %}
public abstract class LookupAdapter implements LookupListener{
  @Override
  public void itemSelected(LookupEvent event){
  }

  @Override
  public void lookupCanceled(LookupEvent event){
  }

  @Override
  public void currentItemChanged(LookupEvent event){
  }
}
{% endhighlight %}

程序里的匿名内部类只监听了itemSelected方法。

{% highlight Java %}
private void showLookup(@NotNull Project project, @NotNull Editor editor) {
 
  editor.getSelectionModel().removeSelection();
  final List<LookupElement> items = createLookupElements();
  final LookupManager lookupManager = LookupManager.getInstance(project);
  final LookupEx lookup = lookupManager.showLookup(editor, items.toArray(LookupElement.EMPTY_ARRAY));
  if (lookup != null) {
    lookup.addLookupListener(new LookupAdapter() {
      @Override
      public void itemSelected(LookupEvent event) {
        final LookupElement item = event.getItem();
        if (item != null) {
          final PsiElement element = myStartElement.getElement();
          final Object object = item.getObject();
          if (element != null && object instanceof ReflectiveSignature) {
            WriteAction.run(() -> applyFix(project, element, (ReflectiveSignature)object));
          }
        }
      }
    });
  }
}
{% endhighlight %}

