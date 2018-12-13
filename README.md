**稻谷丰收**

>此项目是从一个jekyll驱动的GitHub Page博客项目 [My Stack Problems](https://github.com/agusmakmun/agusmakmun.github.io) fork过来的，
>项目中的搜索模块是 [Super Search](https://github.com/chinchang/super-search) 提供。

### 演示
* [https://kksh3ll.github.io](https://kksh3ll.github.io)

#### 功能介绍

* Sitemap and XML Feed
* Pagination in homepage
* Posts under category
* Realtime Search Posts _(title & description)_ by query.
* Related Posts
* Highlight pre
* Next & Previous Post
* Projects page & Detail Project page
* HTML Minify _(Compress HTML)_ using [Jekyll Compress HTML](https://github.com/penibelst/jekyll-compress-html)
* 原项目基础上移除了Disqus comment
* 原项目基础上移除了Share on social media
* 原项目基础上移除了Google analytics

### 安装配置

1. Fork 项目
2. 编辑设置 `_config.yml`
3. 编辑项目文件 `projects.md`, `_data/projects.json`
4. 编辑 `about.md` 文件

### 新建文章

**a. 新建 Category**

在 `category/`文件夹新建文章对应的分类文件, 可参照其中的分类文件

**b. 新建 post 文件**

* 所有文章遵照markdown格式书写，文件后缀必须为 `*.markdown` 或者 `*.md`
* 文章放在 `_posts/` 文件夹下
* 文件名遵照 `<date:%Y-%m-%d>-<slug>.<extension>` 格式命名
