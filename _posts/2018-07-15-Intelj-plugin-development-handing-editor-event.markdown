---
layout: post
title: "Intelj插件开发-处理编辑器事件（Handling Editor Events）"
date: 2018-07-15
categories: [Java, intellij]
---

### 在编辑器中处理击键（Handling keystrokes in the Editor）

#### 1. 实现`TypedActionHandler`接口中的实例

#### 2. 实现处理击键的逻辑

> execute方法包含主要的击键后的处理逻辑，每一次键被压下去都会被调用。

{% highlight Java %}
public class MyTypedHandler implements TypedActionHandler {
    @Override
    public void execute(@NotNull Editor editor, char c, @NotNull DataContext dataContext) {
        final Document document = editor.getDocument();
        Project project = editor.getProject();
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                document.insertString(0, "Typed\n");
            }
        };
        WriteCommandAction.runWriteCommandAction(project, runnable);
    }
}
{% endhighlight %}

#### 3. 设置TypedActionHandler

> 通过TypedActionHandler setupHandler(TypedActionHandler handler)方法创建一个实例，替换原有的TypedActionHandler类。

{% highlight Java %}
public class EditorIllustration extends AnAction {
    static {
        final EditorActionManager actionManager = EditorActionManager.getInstance();
        final TypedAction typedAction = actionManager.getTypedAction();
        typedAction.setupHandler(new MyTypedHandler());
    }
}
{% endhighlight %}

### 使用EditorActionHandler类（Working with EditorActionHandler）

> EditorActionHandler类用在编辑器击键动作后，然后获取EditorActionManager类并通过它执行操作。下面例子使用EditorActionHandler类在当前字符后插入额外的字符。

#### 1. 创建Action并在plugin.xml中注册

{% highlight Java %}
public class EditorHandlerIllustration extends AnAction {
    @Override                                        
    public void actionPerformed(@NotNull AnActionEvent anActionEvent) {
    }
    @Override
    public void update(@NotNull final AnActionEvent anActionEvent) {
    }
}
{% endhighlight %}

{% highlight XML %}
<actions>
    <action id="EditorBasics.EditorHandlerIllustration" class="org.jetbrains.tutorials.editor.basics.EditorHandlerIllustration" text="Editor Handler"
            description="Illustrates how to plug an action in">
      <add-to-group group-id="EditorPopupMenu" anchor="first"/>
    </action>
</action>
{% endhighlight %}

#### 2. 设置可用性（visiblity）

> Action只有在项目打开、编辑器可编辑、在编辑器至少有一个活动字符情况下成立。

{% highlight Java %}
public class EditorHandlerIllustration extends AnAction {
    @Override
    public void actionPerformed(@NotNull AnActionEvent anActionEvent) {
    }
    @Override
    public void update(@NotNull final AnActionEvent anActionEvent) {
        final Project project = anActionEvent.getData(CommonDataKeys.PROJECT);
        final Editor editor = anActionEvent.getData(CommonDataKeys.EDITOR);
        anActionEvent.getPresentation().setVisible((project != null && editor != null && editor.getCaretModel().getCaretCount() > 0));
    }
}
{% endhighlight %}

#### 3. 获取EditorActionHandler类

> 在标准的编辑器定制action，首先需要为action获取EditorActionHandler实例，比如CloneCaretActionHandler实例

{% highlight Java %}
public class EditorHandlerIllustration extends AnAction {
    @Override
    public void actionPerformed(@NotNull AnActionEvent anActionEvent) {
        final Editor editor = anActionEvent.getRequiredData(CommonDataKeys.EDITOR);
        EditorActionManager actionManager = EditorActionManager.getInstance();
        EditorActionHandler actionHandler = actionManager.getActionHandler(IdeActions.ACTION_EDITOR_CLONE_CARET_BELOW);
    }
    @Override
    public void update(@NotNull final AnActionEvent anActionEvent) {
        //...
    }
}
{% endhighlight %}

#### 4. 通过EditorActionHandler执行action

> 为了执行Action我们需要调用EditorActionHandler类中对用的execute方法。
public final void execute(@NotNull Editor editor, @Nullable final Caret contextCaret, final DataContext dataContext)

{% highlight Java %}
public class EditorHandlerIllustration extends AnAction {
    @Override
    public void actionPerformed(@NotNull AnActionEvent anActionEvent) {
        final Editor editor = anActionEvent.getRequiredData(CommonDataKeys.EDITOR);
        EditorActionManager actionManager = EditorActionManager.getInstance();
        EditorActionHandler actionHandler = actionManager.getActionHandler(IdeActions.ACTION_EDITOR_CLONE_CARET_BELOW);
        actionHandler.execute(editor, editor.getCaretModel().getCurrentCaret(), anActionEvent.getDataContext());
    }
    @Override
    public void update(@NotNull final AnActionEvent anActionEvent) {
        //
    }
}
{% endhighlight %}
