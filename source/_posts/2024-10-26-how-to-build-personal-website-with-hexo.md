---
layout: next
title: Hexo搭建博客问题记录
date: 2024-10-26 14:31:14
categories: Hexo
tags: Hexo
---

## Hexo+github homepage搭建个人站点
参考：[2024年，如何使用 github pages + Hexo + Next 搭建个人博客](https://mini-pi.github.io/2024/02/28/how-to-make-blog-wedsite/)

<!-- more -->

## SEO搜索引擎优化
SEO (Search Engine Optimization)，即搜索引擎优化。
简单来说，SEO就是您可以使用提升网站排名的所有方法的总称，SEO用于确保您的网站及其内容在搜索引擎结果页面（SERP）上的可见性。

### 编辑网站信息
编辑`_config.yaml`, 找到`# Site`部分，编辑网站信息
```
# Site
title: PC's Blog
subtitle: Life is short, Play more!
description: "PC's Blog, PC的博客, PC的日志"
keywords: "PC,博客,Blog,Hexo"
author: PC
language: zh-CN
timezone: Asia/Shanghai
```

### 添加网站地图
```
npm install hexo-generator-seo-friendly-sitemap --save
```
Hexo配置文件中添加
```
sitemap:
    path: sitemap.xml
    tag: false
    category: false
```
tag:false, category:false: 网站地图排除标签和分类页面
将sitemap.xml, post-sitemap.xml, page-sitemap.xml 提交后Google Search Console

### 添加robots.txt
`robots.txt`是存放在网站根目录下的一个纯文本文件，可以指定搜索引擎蜘蛛只抓取指定内容，或禁止抓取网站部分内容，可以指定sitemap地址
[robots.txt在线生成工具](https://www.w3cschool.cn/tools/index?name=createrobots)
在source目录下新建robots.txt

### nofollow标签


### 给博客文章生成短链
编辑站点的`_config.yml`(非主题`_config.yml`)
```yaml 
permalink: :layout/:year:month:day:hour:minute:second.html
permalink_defaults:
pretty_urls:
  trailing_index: true # Set to false to remove trailing 'index.html' from permalinks
  trailing_html: true # Set to false to remove trailing '.html' from permalinks
```
注:
* Hexo生成博客文章URL链接时，默认是:year/:month/:day/:title/这样的格式。如果博客文件名有中文的话，URL链接就会包含中文，复制URL路径把它粘贴到其他地方就会把中文变成一大堆乱码，使用不便而且会影响网站的SEO，同时链接层级太多也将影响SEO。
* URL构成越简单越好，百度建议URL不要超过255字节。一个英文字符1字节，一个中文字符2字节。



## 自定义文章底部的版权声明
编辑themes/next/language/zh-CN.yml
```yaml
post: 
  copyright:
    license_content: "所有文章都欢迎转载，注明出处即可."
```


## 参考资料
[https://mini-pi.github.io/2024/02/28/how-to-make-blog-wedsite](https://mini-pi.github.io/2024/02/28/how-to-make-blog-wedsite)
[https://www.duheweb.com/post/20210414222449.html](https://www.duheweb.com/post/20210414222449.html)
[https://blog.dejavu.moe/posts/hexo-seo/](https://blog.dejavu.moe/posts/hexo-seo/)
