---
title: "使用hugo搭建blog"
date: 2020-10-10T20:36:06+08:00
draft: true
categories:
 - tools
tags:
 - blog
 - hugo
toc: true
---

# 使用hugo搭建blog

## Install

### MacOS下安装

```shell
$ brew install hugo
```

## Usage

### Create New Blog

```shell
$ hugo new site $your_blog_name
```

### Install theme

```shell
$ cd $your_blog_name
$ git init
# 作为子模块方便管理和更新
$ git submodule add https://github.com/MunifTanjim/minimo.git themes/minimo
# 

```

如果之后有对主题进行自定义修改，又想忽略掉这个子模块的git变动，可以修改.gitmodule文件，加上ignore = dirty

```
[submodule "themes/minimo"]
    path = themes/minimo
    url = https://github.com/MunifTanjim/minimo
    ignore = dirty
```

#### minimo主题配置

```shell
# 如果要使用某个主题，最好使用该主题的example的配置，再在上面进行修改定制
$ cp themes/minimo/exampleSite/config.toml .
# staticman是用来给静态网站增加评论功能的，如果要用评论功能需要配置这个
$ cp themes/minimo/exampleSite/staticman.toml .
# 为了省事直接把这两个也先copy过来了
$ cp -r themes/minimo/exampleSite/content .
$ cp -r themes/minimo/exampleSite/data .
```

如果要在上方的菜单栏配置posts/categories/tags，可以在content目录下新建对应文件夹，之后创建一个_index.md的文件，添加下面的内容

```markdown
---
date: 2017-09-28T08:00:00+06:00
title: Posts 
linkTitle: Posts 
slug: posts
menu: main
weight: -290
---
```

##### 添加百度统计

在head模板的`</head>`标签之前放入百度统计生成的`<script>...</script>`代码就行了

hugo只需要在根目录的文件夹创建和主题同路径同名的文件就可以覆盖并生效，所以不需要修改主题的模板文件

```shell
# minimo主题的head模板文件是这个/themes/minimo/layouts/partials/head/head.html
$ mkdir -p layouts/partials/head/
$ cp themes/minimo/layouts/partials/head/head.html layouts/partials/head/head.html
$ vim layouts/partials/head/head.html
```

### create new post

一定要加上.md后缀
```shell
$ hugo new posts/first_blog.md
```

### Debug

可以先在本地进行调试看看效果，执行之后在localhost:1313就可以访问了

```shell
$ hugo server -D
```

### 生成静态网站

执行这条指令之后就会生成一个public文件夹，静态的网站就在这里面

```shell
$ hugo -D
```

### 上传到Github Pages

#### Create GitHub repo

先在GitHub上创建两个repo，一个是github pages的，格式必须是xudai3.github.io这样，另外一个随便取名叫blog

xudai3.github.io仓库用来存放public文件夹下的静态网站，上传之后就可以访问https://xudai3.github.io了

而blog仓库可以用来存放这整个的hugo文件夹

#### Upload

可以在hugo项目目录下的.gitignore里把public文件夹忽略掉，然后`git submodule add -f git@github.com:xudai3/xudai3.github.io.git public`把public作为一个git子模块分开管理

如果之前生成过public文件夹，执行这个之前要先删除掉public文件夹否则执行不了

还有因为之前被忽略了，要加上-f强制执行才能执行成功

之后就按照正常的git上传流程就行了

#### Travis-ci