---
layout: post                          # (require) default post layout
title: "Your Title"                   # (require) a string title
date: 2016-04-20 19:51:02 +0700       # (require) a post date
categories: [python, django]          # (custom) some categories, but makesure these categories already exists inside path of `category/`
tags: [foo, bar]                      # (custom) tags only for meta `property="article:tag"`
image: Broadcast_Mail.png             # (custom) image only for meta `property="og:image"`, save your image inside path of `static/img/_posts`
---

Jekyll 支持插件功能，你可以很容易的加入自己的代码。

## 安装插件

在网站根下目录建立 `_plugins` 文件夹，插件放在这里即可。 Jekyll 运行之前，会加载此目录下所有以
 `*.rb` 结尾的文件。

通常，插件最终会被放在以下的目录中：

1. Generators
2. Converters
3. Tags

## 转换器

如果想使用一个新的标记语言，可以用你自己的转换器实现，Markdown 和 Textile 就是这样实现的。


下边的例子实现了一个转换器，他会用 `UpcaseConverter` 来转换所有以 `.upcase` 结尾的文件。

{% highlight ruby %}
module Jekyll
  class UpcaseConverter < Converter
    safe true
    priority :low

    def matches(ext)
      ext =~ /^\.upcase$/i
    end

    def output_ext(ext)
      ".html"
    end

    def convert(content)
      content.upcase
    end
  end
end
{% endhighlight %}

转换器需要最少实现以下 3 个方法：



在上边的例子中， `UpcaseConverter#matches` 检查文件后缀名是不是 `.upcase` ;
`UpcaseConverter#convert` 会处理检查成功文件的内容，即将所有的字符串变成大写；最终，保存的结果
将以作为后缀名 `.html` 。

## 标记

如果你想使用 liquid 标记，你可以这样做。 Jekyll 官方的例子有 `highlight` 和 `include` 等
标记。下边的例子中，自定义了一个 liquid 标记，用来输出当前时间：

{% highlight ruby %}
module Jekyll
  class RenderTimeTag < Liquid::Tag

    def initialize(tag_name, text, tokens)
      super
      @text = text
    end

    def render(context)
      "#{@text} #{Time.now}"
    end
  end
end

Liquid::Template.register_tag('render_time', Jekyll::RenderTimeTag)
{% endhighlight %}

liquid 标记最少需要实现如下方法：


你必须同时用Liquid模板引擎注册自定义标记，比如：：

{% highlight ruby %}
Liquid::Template.register_tag('render_time', Jekyll::RenderTimeTag)
{% endhighlight %}

对于上边的例子，你可以把如下标记放在页面的任何位置：

{% highlight ruby %}
{% raw %}
<p>{% render_time page rendered at: %}</p>
{% endraw %}
{% endhighlight %}

我们在页面上会得到如下内容：

{% highlight html %}
<p>page rendered at: Tue June 22 23:38:47 –0500 2010</p>
{% endhighlight %}

### Liquid 过滤器

你可以像上边那样在 Liquid 模板中加入自己的过滤器。过滤器会把自己的方法暴露给 liquid 。所有的方法
都必须至少接收一个参数，用来传输入内容；返回值是过滤的结果。

{% highlight ruby %}
module Jekyll
  module AssetFilter
    def asset_url(input)
      "http://www.example.com/#{input}?#{Time.now.to_i}"
    end
  end
end

Liquid::Template.register_filter(Jekyll::AssetFilter)
{% endhighlight %}



### Flags

当写插件时，有两个标记需要注意：


已上边例子的插件为例，应该这样设置这两个标记：

{% highlight ruby %}
module Jekyll
  class UpcaseConverter < Converter
    safe true
    priority :low
    ...
  end
end
{% endhighlight %}

<div class="note info">
  <h5>期待你的作品</h5>
  <p>
    如果你有一个 Jekyll 插件并且愿意加到这个列表中来，可以<a href="../contributing/">阅读此须知</a>，并参照着来做。
  </p>
</div>