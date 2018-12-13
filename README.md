**稻谷丰收**

>此项目是从一个jekyll驱动的GitHub Page博客项目[My Stack Problems](https://github.com/agusmakmun/agusmakmun.github.io)fork过来的，>项目中的搜索模块是 [Super Search](https://github.com/chinchang/super-search)提供。

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

1. Fork this repository
2. Edit site settings inside file of `_config.yml`
3. Edit your projects at file of `projects.md`, `_data/projects.json` and inside path of `_project/` _(for detail project)_.
4. Edit about yourself inside file of `about.md`

### 新建文章

**a. Add new Category**

All categories saved inside path of `category/`, you can see the existed categories.

**b. Add new Posts**

* All posts bassed on markdown syntax _(please googling)_. allowed extensions is `*.markdown` or `*.md`.
* This files can found at the path of `_posts/`.
* and the name of files are following `<date:%Y-%m-%d>-<slug>.<extension>`, for example:
