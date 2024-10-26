---
layout: next
title: Hexo搭建博客的常见问题
date: 2024-10-26 14:31:14
categories: Hexo
tags: Hexo
---

## Hexo+github homepage搭建个人站点
参考：[2024年，如何使用 github pages + Hexo + Next 搭建个人博客](https://mini-pi.github.io/2024/02/28/how-to-make-blog-wedsite/)

<!-- more -->

## 自定义文章底部的版权声明
编辑themes/next/language/zh-CN.yml
```yaml
post: 
  copyright:
    license_content: "所有文章都欢迎转载，注明出处即可."
```
## 给博客文章生成短链
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

## 参考资料
[https://mini-pi.github.io/2024/02/28/how-to-make-blog-wedsite](https://mini-pi.github.io/2024/02/28/how-to-make-blog-wedsite)
[https://www.duheweb.com/post/20210414222449.html](https://www.duheweb.com/post/20210414222449.html)
