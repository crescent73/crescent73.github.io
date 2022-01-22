---
title: 搭建hexo博客 & github部署 & 使用butterfly主题
date: 2022-01-17 15:19:13
categories: 
- blogs
tags:
- hexo
- github
- butterfly
---

# hexo
hexo是一个由node.js编写的博客框架，使用markdown解析文章，可以生成静态网页部署在github上。  
hexo官网指路：[链接](https://hexo.io/zh-cn/)  
## hexo安装
hexo安装的前提需要安装git和node.js。  
必备的应用程序安装完成以后可以使用npm安装：  
`$ npm install -g hexo-cli`

## 新建hexo项目
hexo安装完成以后通过以下命令初始化hexo项目
``` shell
$ hexo init <folder>
$ cd <folder>
$ npm install
```
新建完成后，文件目录如下：
```
.
├── _config.yml 配置文件
├── package.json
├── scaffolds 模板文件
├── source 博客文件
|   ├── _drafts
|   └── _posts
└── themes 主题文件
```
## 启动hexo服务
执行以下命令，访问 `http://localhost:4000/` 就能看到博客了
``` shell
$ hexo generate # 生成静态文件
$ hexo server  # 启动服务器
```

# 使用GitHub Pages搭建hexo博客
1. 在github上创建一个名称为 `用户名.github.io` 的仓库。
2. 给hexo项目安装hexo-deployer-git插件：`npm install hexo-deployer-git --save`
3. 修改 `_config.yml` 文件末尾的deploy部分
``` yml
deploy:
  type: git
  repository: git@github.com:用户名/用户名.github.io.git
  branch: master
```
4. 完成后执行 `hexo deploy` 指令，将网站部署到GitHub Pages
5. 访问 `https://用户名.github.io` 即可看到hexo网站

附：github仓库里存储的是编译后的网页内容，如果想保存hexos的源码，可以给仓库新建一个分支，上传源码。

# hexo使用butterfiy主题
默认的主题太丑了，找了半天主题，最后选择了 [butterfly主题](https://butterfly.js.org/) ,原因当然是，够好看。 
## 安装主题 
主题安装指令：`npm i hexo-theme-butterfly`。注意通过 `npm` 安装会在 `node_modules` 生成主题文件夹。
## 应用主题
在 `_config.yml` 中修改theme为butterfly即可。`theme: butterfly`
## 安装插件
安装style和stylus渲染器插件
`npm install hexo-renderer-pug hexo-renderer-stylus --save`
## 主题设置
在 hexo 的根目录创建一个文件 `_config.butterfly.yml`，并把主题目录的 `_config.yml` 内容复制到 `_config.butterfly.yml` 去。
> 注意：  
> 1. 是主题目录中的 `_config.yml` 文件，不是根目录的 `_config.yml` 文件。  
> 2. 以后在 `_config.butterfly.yml` 中修改主题配置即可
> 3. Hexo会自动合并主题中的 `_config.yml` 和 `_config.butterfly.yml` 里的配置，如果存在同名配置，会使用 `_config.butterfly.yml` 的配置，其优先度较高。  

详细的配置内容可以查看butterfly的[主题配置文档](https://butterfly.js.org/categories/Docs%E6%96%87%E6%AA%94/)

贴一下我的主题配置文件：
``` yml
menu:
  首页: / || fas fa-home
  时间轴: /archives/ || fas fa-archive
  标签: /tags/ || fas fa-tags
  分类: /categories/ || fas fa-folder-open
# Code Blocks (代碼相關)
# --------------------------------------
highlight_theme: darker #  darker / pale night / light /
# Image (圖片設置)
# --------------------------------------
# Favicon（網站圖標）
favicon: /img/favicon.png
# Avatar (頭像)
avatar:
  img: /img/avatar.jpg
  effect: false
# The banner image of home page
index_img: /img/bg.jpg
# If the banner of page not setting, it will show the top_img
default_top_img: /img/bg.jpg
cover:
  # display the cover or not (是否顯示文章封面)
  index_enable: false
  aside_enable: false
  archives_enable: false

# wordcount (字數統計)
wordcount:
  enable: true
  post_wordcount: true
  min2read: true
  total_wordcount: true

# the subtitle on homepage (主頁subtitle)
subtitle:
  enable: true
  # Typewriter Effect (打字效果)
  effect: true
  # loop (循環打字)
  loop: true
  # subtitle 會先顯示 source , 再顯示 sub 的內容
  source: false
  # 如果關閉打字效果，subtitle 只會顯示 sub 的第一行文字
  sub:
    - Never put off till tomorrow what you can do today
    - Drink more hot water
```

# 新建blog
基本指令是：`$ hexo new [layout] <title>`，layout一般由post和page，具体的区别不太清楚，默认是post布局。  
如果想要制定文章路径，那么使用以下指令: `hexo new [layout] --path path/<title> <title>`  
此外，还可以新建文章模板，在scaffolds文件夹中新建一个md文件，里面填写你设置的模板内容。注意在模板文件开头要加上 `layout: name` 在新建文章时，指定layout为自定义的模板名称就行了。