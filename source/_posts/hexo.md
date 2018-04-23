---
title: Hexo+Github搭建博客
date: 2018-02-12
categories: Hexo+Github搭建博客
tags: 
- github
- hexo
- blog

---

本帖子记录的是使用Hexo配合Github，利用github提供的域名来搭建自己的私人博客


## 安装Node.js

### 下载

用wget命令下载最新版(或指定版本)的Node.js

``` bash
$ wget https://nodejs.org/dist/v9.3.0/node-v9.3.0-linux-x64.tar.xz
```

最新版本请查看[下载地址](https://nodejs.org/dist)



### 解压文件

``` bash
$ xz -d node-v9.3.0-linux-x64.tar.xz  //解压为tar文件
$ tar -xvf node-v9.3.0-linux-x64.tar
```

或

``` bash
$ tar -xvJf node-v9.3.0-linux-x64.tar.xz
```

然后移动文件夹

``` bash
$ mv  node-v9.3.0-linux-x64 node-v9.3.0
$ mv  node-v9.3.0 /usr/local/node
```


### 配置环境变量

``` bash
## 编辑 /etc/profile (使用vim)
$ vim /etc/profile
## 在底部添加 PATH 变量
export PATH=$PATH:/usr/local/node/bin
# 保存退出，先按exit键，再按shift+：
:wq
## 最后保存并使其生效即可
$ source /etc/profile
```



## 安装Hexo

### 安装Hexo主程序

自行选择安装目录，这里我直接安装在自己用户的家目录下

``` bash
## 创建目录
$ mkdir -p ~/hexo

## 切换目录
$ cd ~/hexo

## 安装Git(已安装可跳过)
$ yum install -y git git-core

## 安装 Hexo
$ npm install -g hexo-cli

## 初始化Hexo
$ hexo init
```



### 安装Hexo相关插件

``` bash
$ npm install hexo-generator-index --save
$ npm install hexo-generator-archive --save
$ npm install hexo-generator-category --save
$ npm install hexo-generator-tag --save
$ npm install hexo-server --save
$ npm install hexo-deployer-git --save
$ npm install hexo-deployer-heroku --save
$ npm install hexo-deployer-rsync --save
$ npm install hexo-deployer-openshift --save
$ npm install hexo-renderer-marked --save
$ npm install hexo-renderer-stylus --save
$ npm install hexo-generator-feed --save
$ npm install hexo-generator-sitemap --save
```


### Hexo配置文件模板

Hexo配置文件采用的是yaml语法格式，若修改请按照语法格式修改

模板例子：

```bash
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: Hexo
subtitle:
description:
author: Jayden
language:
timezone:

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://yoursite.com
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

# Directory
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: true
relative_link: false
future: true
highlight:
  enable: true
  line_number: true
  auto_detect: false
  tab_replace:
  
# Home page setting
# path: Root path for your blogs index page. (default = '')
# per_page: Posts displayed per page. (0 = disable pagination)
# order_by: Posts order. (Order by date descending by default)
index_generator:
  path: ''
  per_page: 10
  order_by: -date
  
# Category & Tag
default_category: uncategorized
category_map:
tag_map:

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10
pagination_dir: page

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
#theme: landscape
theme: archer

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:Polaris0112/Polaris0112.github.io.git
  branch: master
  message: '站点更新:{{now("YYYY-MM-DD HH/mm/ss")}}'

## archer
jsonContent:
  meta: true
  pages: false
  posts:
    title: true
    date: true
    path: true
    text: false
    raw: false
    content: false
    slug: false
    updated: false
    comments: false
    link: false
    permalink: true
    excerpt: false
    categories: true
    tags: true
```

这个模板例子我使用的是archer，需要安装主题模板才能使用，该主题的[Github地址](https://github.com/fi3ework/hexo-theme-archer)

安装命令为

```bash
## 在Hexo目录下执行
$ npm install hexo-generator-json-content --save && git clone https://github.com/fi3ework/hexo-theme-archer.git themes/archer

## 修改Hexo目录下的 _config.yml 的 theme 字段为 archer
theme: archer
```

[更多主题](https://hexo.io/themes/index.html)

关于主题更换，可以从以上网址进入对应的Github仓库，都有详细的更换主题说明。



## 接入Github

先在Github上注册一个账号，然后在Setting相关的地方设置`SSH and GPG key`添加自己的公钥，在服务器保留私钥。

设置好后自己可以先创建一个测试仓库试试clone和push，没问题再进行下一步。


### 创建博客项目

创建一个新项目，项目名称为 用户名.github.io ，比如我的Github用户名是polaris0112，则创建的项目名为polaris0112.github.io

然后在Github该仓库中绿色按钮`Clone or download`选择`Clone with SSH`复制对应git uri。

编辑Hexo目录下的_config.yml

```bash
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: git@github.com:Polaris0112/Polaris0112.github.io.git     ##这一行，复制替换上一步得到的git uri。
  branch: master
  message: '站点更新:{{now("YYYY-MM-DD HH/mm/ss")}}'
```
这里，repo就是你创建的博客项目，以后修改之后就是上传到这个仓库然后给外网访问。


最后一步，编译，上传静态代码

```bash
## 编译
hexo generate //可以简写 hexo g
## 在主机的hexo目录下 执行以下命令将自动更新到Github
hexo deploy //可以简写 hexo d
```

如果 hexo d 报错

```bash
ERROR Deployer not found:

## 运行
$ npm install --save hexo-deployer-git
```

## 其它命令

``` bash
##清空你hexo/public里的内容，可以在每次修改配置文件后清除一下
$ hexo clean

## 多条命令可以一块输入
hexo clean && hexo g && hexo d
```

这样就配置好了，可以使用https://polaris0112.github.io/ （使用你自己的域名）来访问。
