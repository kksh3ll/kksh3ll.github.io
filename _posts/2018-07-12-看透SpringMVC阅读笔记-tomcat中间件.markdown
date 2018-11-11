---
layout: post
title: 看透SpringMVC阅读笔记-tomcat中间件
data: 2018-06-26
categories: Java
---

## Tomcat的顶层结构

Tomcat的顶层容器叫Server，代表整个服务器。Server至少包含一个service，service包含Connector和Containner。

- Connector处理连接相关的事情，提供socket和request、response的转换，可以有多个

- Containner用于封装管理Servlet，具体处理request的请求，只能有一个

![img](/img/tomcat20180626.jpeg)

org.apache.catalina.startup.Catalina类是整个tomcat的管理类，有load、start、stop分别管理整个服务器的生命周期，load方法根据conf/server.xml文件创建Server并调用Server的init方法初始化。

Catalina中的await方法直接调用Server的await方法进入循环，让主线程不会退出。

Tomcat的入口main方法在org.apache.catalina.startup.Bootstrap中，Bootstrap的作用类似一个CatalinaAdaptor，具体处理过程还是由Catalina来完成，这样做的好处是把启动的入口和具体的管理类分开，从而更方便的创建出多种启动方式，每种启动方式只需要写一个相应的CatalinaAdaptor就可以。

## Tomcat的生命周期管理

Tomcat通过`org.apache.catalina.Lifecycle`接口统一管理生命周期，所以组建的生命周期都要实现`Lifecycle`接口，`Lifecycle`总共做四件事：

- 定义13个String类型的常量，用于`LifecycleEvent`事件的type属性

- 定义3个管理监听器的方法`addLifecycleListener`、`findLifecycleListeners`、`removeLifecycleListener`，用来添加查找和删除`LifecycleListener`类型的监听器

- 定义4个生命周期的方法init、start、stop、destroy，用于执行生命周期阶段的操作

- 定义获取当前状态的2个方法`getState`和`getStateName`，`getState`返回值`LifecycleState`是枚举类型，里边列举了生命周期的各个节点

`org.apache.catalina.util.LifecycleBase`类为`Lifecycle`接口提供了默认实现：监听器管理是专门使用了`LifecycleSuppport`类来完成的；生命周期方法`init`、`start`、`stop`、`destroy`中设置了相应的状态并调用相应的模版方法`initInternal`、`startInternal`、`stopInternal`、`destroyInternal`，模版方法由子类具体实现

## Containner分析

Containner一共有4个子接口Engine、Host、Context、Wrapper和一个默认实现类ContainnerBase，每个子接口都是一个容器，这4个字容器有一个对应的StandardXXX实现类，并且这些实现类都继承ContainnerBase类。通常使用的Servlet封装在Wrapper中。

![img](/img/containner20180626.jpg){:width="800"}

- Engine管理多个Host，一个Service只能有一个Engine

- 每个Host代表一个虚拟主机

- 每个Context代表一个应用

- 每个Wrapper封装着一个Servlet

Containner的启动是通过`init`和`start`方法来完成的，这俩个方法会在tomcat起动时被Service调用，Containner也是按照tomcat的生命周期来管理的，init和start方法调用initInternal和startInternal方法来具体处理，但Containner和tomcat整体结构启动的方式稍微不一样：

- Containner4个子容器的共同父类`ContainnerBase`定义了Containner容器的`initInternal`和`startInternal`方法通用处理内容

- 除了最顶层容器的init是被Service调用，子容器的init方法不是在容器中逐层循环调用，而是在执行Start方法后通过状态判断没有初始化后调用

- Start方法除了在父容器中的`startInternal`中调用，还会在父容器的添加子容器的addChild中调用，因为`Context`和`Wrapper`是动态添加的（我们在站点目录下放一个文件夹活着war包就可以添加一个context，在web.xml中配置一个Servlet就可以添加一个Wrapper），所以Context和Wrapper是在容器启动过程中动态查找并添加到相应的父容器中的。

ContainnerBase的initInternal方法主要初始化ThreadPoolExecutor类型的startStopExecutor属性，用于管理启动和关闭线程，代码如下：

{% highlight Java %}
protected void initInternal() throws LifecycleException {
    BlockingQueue<Runnable> startStopQueue = new LinkedBlockingQueue<>();
    startStopExecutor = new ThreadPoolExecutor(
      getStartStopThreadsInternal(), 
      getStartStopThreadsInternal(), 10L, TimeUnit.SECONDS, 
      startStopQueue, 
      new StartStopThreadFactory(getName() + "-startStop-"));
    startStopExecutor.allowCoreThreadTimeOut(true);
    super.initInternal();
}
{% endhighlight %}

`ThreadPoolExecutor`继承自Executor用于管理线程，特别是Runable类型的线程。

ContainnerBase的startInternal方法主要做5件事

- 如果有`Cluster`和`Realm`则调用start方法（Cluster用于配置集群，Realm是tomcat的安全域，管理资源的访问权限）

- 调用所以子容器的start方法启动子容器

- 调用管道中的`Value`的`start`方法来启动管道

- 启动完成后将生命周期设置为`LifecycleState.STARTING`状态

- 启动后台线程定期做一些事情

## Connector分析

`Connector`用于接收请求并将请求封装成`Request`和`Response`来具体处理，最底层是使用Socket连接，`Request`和`Response`是按照HTTP协议来封装，所以`Connctor`同时实现了`TCP/IP`协议和`HTTP`协议，`Request`和`Response`封装完以后交给`Containner`处理，`Containner`是Servlet容器，`Containner`处理完交给`Connector`，最后`Connector`使用Socket将处理结果返回给客户端，整个请求就处理完了。

Connctor中具体是用ProtocolHandler处理请求的，不同的ProtocolHandler代表不同的连接类型，比如`Http11NioProtocol`是使用NioSocket来连接的。

ProtocolHandler有3个重要的组件：

- `EndPoint` 用于处理底层Socket的连接

- `Processor` 用于将EndPoint接收的Socket封装成Request

- `Adapter` 用于将封装好的Request交给Containner进行具体处理

![img](/img/conncetor20180626.jpg)

`ProtocolHandler`接口的抽象实现类`AbstractProtocol`分2种类型`Ajp`和`HTTP`，默认配置中的`org.apache.coyote.http11.Http11NioProtocol`使用HTTP1.1协议，TCP层使用`NioSocket`来传输数据。

Http11NioProtocol构造函数中创建NioEndPoint类型的EndPoint，并新建Http11ConnectionHandler类型的Handler然后设置到EndPoint中。

### 处理TCP/IP协议的EndPoint

EndPoint用于处理具体连接和传输数据，NioEndPoint继承自`org.apache.tomcat.util.net.AbstractEndPoint`，在EndPoint中新增Poller和SocketProcessor内部类，NioEndPoint的init和start方法在父类AbstractEndPoint中，主要调用模板方法bind和startInternal，这俩个方法在NioEndPoint中具体实现

### 处理HTTP协议的Processor

Processor用于处理应用层HTTP协议，Http11Processor类的process方法定义在其父类AbstractProcessorLight中，它会接着调用service抽象方法处理请求，Http11Processor类实现了自己的service方法

### 适配器Adapter

Adapter只有一个实现类org.apache.catalina.connector包下的CoyoteAdapter类。Http11Processor类的service方法会调用Adapter的service方法来调用Containner管道中的invoke方法处理请求，将原来创建的org.apache.coyto包下的Request和Response封装成org.apache.catalina.connector的Request和Response

{% highlight Java %}
connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);
{% endhighlight %}




