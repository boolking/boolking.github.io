---
title: "静态网站生成工具选择"
date: 2020-12-21T15:18:17+08:00
categories: ['hugo']
tags: ['jekyll', 'hexo', 'hugo']
draft: false
---


TLDR:

考察了一下static web site generator，常用的有jekyll，hexo和hugo，我根据自己的情况，选择了Hugo。
|Name|Home|Language|Pro|Con|
|----|----|--------|---|---|
|hugo|https://gohugo.io/|golang|安装简单</br>github star数多</br>有问题可以自己解决|需要使用github action更新|
|jekyll|https://jekyllrb.com/|ruby|老牌工具</br>github star数最多</br>github pages原生支持</br>模板比较多|需要安装ruby</br>速度慢</br>介绍比较旧|
|hexo|https://hexo.io/|JavaScript|中文文档比较好|需要使用github action更新|

-----------------------------

## Jekyll
1. github pages官方文档中使用的是jekyll，之前ruby也没怎么用过，虽然与其他语言大同小异，但是用起来不够熟悉，导致很多步骤出现问题。
2. 然后网上关于jekyll的文章很多都是4-5年前的。
3. github pages的版本是3.9.0，官方已经是4.2.0了。
4. jekyll官方还不支持windows，考虑到以后可能不止在一个平台写日志，这个已经是一个不小的缺点了。
5. 安装gem和bundle的官方源太慢了，虽然可以替换国内mirror解决，第一次安装的时候停了好长时间才发现是网络问题，在心中又减了一分。最后，看源码和查找问题比较麻烦（还得先学ruby），果断放弃。

## Hexo
1. Hexo是JavaScript的，虽然nodejs也经常用，但是自从用上golang和python后，已经尽量避免使用nodejs了，放弃。
2. Hexo的中文文档还是不错的。


## Hugo
Hugo在star数量上跟jekyll相差不大，而且是golang实现，下载的时候官方release就一个可执行文件，安装太方便了，本来写blog就是要简单，如果环境搭建太麻烦就得不偿失了。

下面就详细讲一下Hugo的使用。

### 安装
在[github release](https://github.com/gohugoio/hugo/releases)页面下载安装包，解压其中的可执行文件，将其路径加入到path中

### 新建site
命令行输入
```bash
# 新建site
hugo new site boolking.github.io
cd boolking.github.io

#初始化git
git init .
git add remote https://github.com/boolking/boolking.github.io
```

### 选择theme
到[Hugo Themes](https://themes.gohugo.io/)选择一款主题，我使用的是第一个[Hugo Future Imperfect Slim](https://themes.gohugo.io/hugo-future-imperfect-slim/)
```bash
cd themes
git submodule add https://github.com/pacollins/hugo-future-imperfect-slim.git
```

### 设置config.toml
参考[wiki](https://github.com/pacollins/hugo-future-imperfect-slim/wiki/config.toml)，修改自己的config.toml

### 启动local server
启动local server，打开browser，边编辑边看效果
```bash
hugo server -D
```
