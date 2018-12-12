---
layout: post
title:  Java的反射机制
date:   2016-05-11 20:23:12
categories: [Java]
---

### 反射机制的原理
 反射机制是指程序的在运行过程中，对于一个类能知道这个类的所有属性和方法。这种动态获取信息的功能称之为java的反射机制。
 反射机制主要有一下功能：

 * 运行时判断一个对象的所属的类，调用任意一个对象的方法
 * 运行时判断一个类的成员变量和方法，构造一个类的对象
 * 生成动态代理

### 反射机制的相关API

 * 获得对象的包名和类名

  {% highlight Java %}
  package com.github.test;
  public class Test {
    public static void main(String[] args){
      Test test = new Test();
      System.out.println(test.getClass().getName());
    }
  }
  {% endhighlight %}

 * 实例化一个类的对象
 {% highlight Java %}
 public static void main(String[] args) throws Exception {
   Class<?> class1 = null;
   class1 = Class.forName("com.github.test.Test");
   Constructor<?> cons[] = class1.getConstructors();
     // 查看每个构造方法需要的参数
     for (int i=0;i<cons.length;i++){
         Class<?> clazzs[] = cons[i].getParameterTypes();
         for (int j=0;j<clazzs.length;j++) {
             if (j == clazzs.length - 1)
                 System.out.print(clazzs[j].getName());
             else
                 System.out.print(clazzs[j].getName() + ",");
         }
         System.out.println(")");
     }
 }
 {% endhighlight %}

 * 获取某各类的属性和方法
  {% highlight Java %}
  public static void main(String[] args){
    Class<?> clazz = Class.forName("com.github.test.Test");
    Field [] Field = clazz.getDeclaredFiled();
    for (int i=0;i<field.length;i++) {
      Class<?> type = field[i].getType();
      System.out.println(type.getName() + " " + field[i].getName());
  }
  {% endhighlight %}

 {% highlight Java %}
 public static void main(){
   Class<?> clazz = Class.forName("com.gethub.test.Test");
   Method method[] = clazz.getMethods();
   for (int i=0;i<method.length;i++){
     System.out.println(method[i].getName());
   }
 }
 {% endhighlight %}

 * 反射机制的动态代理
 通过反射来生成一个代理，分三步：
 1. new出代理对象，通过实现InvacationHandler接口，然后new出代理对象来。
 2. 通过Proxy类中的静态方法newProxyInstance，被代理的对象实例。
 3. 执行方法，代理成功。
 {% highlight Java %}
 class MyInvocationHandler implements InvocationHandler {
    private Object obj = null;
    public Object bind(Object obj) {
        this.obj = obj;
        return Proxy.newProxyInstance(obj.getClass().getClassLoader(),
         obj.getClass().getInterfaces(), this);
    }
    public Object invoke(Object proxy, Method method, Object[] args){
        Object temp = method.invoke(this.obj, args);
        return temp;
    }
 }
 {% endhighlight %}
