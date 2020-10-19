---
title: "使用hugo搭建blog"
date: 2020-10-10T20:36:06+08:00
lastmod: 2020-10-10T20:36:06+08:00
slug: hugo-build-blog
categories:
 - tools
tags:
 - blog
 - hugo
toc: true
draft: true
---

# 使用hugo搭建blog

## Install

### MacOS下安装

```shell
$ brew install hugo
```

## Usage

### 新建项目

```shell
$ hugo new site $your_blog_name
```

### 安装主题

```shell
$ cd $your_blog_name
$ git init
# 作为子模块方便管理和更新
$ git submodule add https://github.com/MunifTanjim/minimo.git themes/minimo
```

如果之后有对主题进行自定义修改，又想忽略掉这个子模块的git变动，可以修改.gitmodule文件，加上ignore = dirty，不过hugo只需要在根目录的文件夹创建和主题同路径同名的文件就可以覆盖并生效，所以不需要修改主题的模板文件

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

```shell
# minimo主题的head模板文件是这个/themes/minimo/layouts/partials/head/head.html
$ mkdir -p layouts/partials/head/
$ cp themes/minimo/layouts/partials/head/head.html layouts/partials/head/head.html
$ vim layouts/partials/head/head.html
```

### 创建文章

一定要加上.md后缀
```shell
$ hugo new posts/first_blog.md
```

### 调试

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

#### 新建GitHub仓库

先在GitHub上创建两个repo，一个存放原始hugo项目的，另一个是存放github pages静态网页文件的，repo名字格式必须是xudai3.github.io这样，把public文件夹下的生成的静态网页文件上传之后到这个仓库之后就可以访问https://xudai3.github.io了

#### 手动上传到GitHub

可以在hugo项目目录下的.gitignore里把public文件夹忽略掉，然后`git submodule add -f git@github.com:xudai3/xudai3.github.io.git public`把public作为一个git子模块分开管理

如果之前生成过public文件夹，执行这个之前要先删除掉public文件夹否则执行不了

还有因为之前被忽略了，要加上-f强制执行才能执行成功

之后就按照正常的git上传流程就行了

#### hugo集成Travis CI

之前两个仓库分开管理比较麻烦，可以通过Travis CI管理blog项目，检测到更新后自动生成静态网页文件上传到

xudai3.github.io

##### 生成GitHub token

GitHub->Settings->Developer Settings->Personal access tokens或者直接访问https://github.com/settings/tokens

然后点击generate new token，勾选*public_repo, repo:status, repo_deployment*这三个权限就够了

##### 配置travis ci

访问https://travis-ci.org/，点击右上角登陆，选择GitHub账号登陆，选择需要监听的GitHub仓库

选择blog项目之后，点击setting，配置一个环境变量GITHUB_TOKEN，value填刚才生成的GitHub token，之后编写.travis.yml的时候就可以使用这个变量

##### 编写.travis.yml

```yml
# 需要先下载安装hugo
install:
  - wget https://github.com/gohugoio/hugo/releases/download/v0.76.3/hugo_0.76.3_Linux-64bit.deb
  - sudo dpkg -i hugo*.deb
  - hugo version

before_script:
  - git config --global user.name "xudai3"
  - git config --global user.email "xudai3@qq.com"

# 指定要执行的脚本
script:
  - hugo -D
  
# 先clone下来清空非隐藏文件，在把新生成的public文件内容复制进去，这样可以保留commit记录
after_success:
  - git clone https://${GITHUB_TOKEN}@github.com/xudai3/xudai3.github.io.git container
  - rm -rf container/*
  - cp -r public/* container
  - cd container
  - git add .
  - git commit -m "Travis update blog"
  - git push -u origin master -f
```

##### 上传到blog仓库

之后每次push到blog仓库之后就会触发Travis了