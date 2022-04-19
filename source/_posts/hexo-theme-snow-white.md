---
title: 给博客上了一款新主题 《snow white》
tags: 技术
date: 2022-04-16 08:00:00
keyword: hexo snow white theme, snow white theme, hexo theme, hexo
description: snow white 是 hexo 的一款极致简约的博客主题，纯白色调设计，无任何多余元素和图标，seo 支持友好，如果喜欢简约风格可以尝试
---

使用 hexo 创建博客也有好几个年头了，可已写下的文章的不多，所以就一直是默认的主题挂在那儿。

今年以来立志要练习写作，也就打算好好的整一下这个静态博客了，第一件事自然是得整个好看的、符合我极简审美的主题。耐何在 Hexo 主题站上找了一圈，也没有找到完全合意的主题，所以就想自己写一个。说干就开，也就诞生了这个主题，起名叫《snow white》。

这是我的第一个主题，虽有些不足，但也有些特点。

###　极致的简约

整体以白色设计为主，没有边边框框、背景色之类的装饰，自然更不会有 icon 和各种图标了。

### 使用 Tailwind css

主题的风格样式完全使用 [Tailwind css](https://tailwindcss.com/) 进行定义，编译后也仅仅只有 13kb。Tailwind css　是一个功能类优先的 CSS 框架，样式能够语义化，这样整体的布局、样式都更加清晰明了，方便进行二次开发和调整。

### 中文排版

中文排版一直是个比较头痛的问题了，为使文章易于阅读，在本主题中直接使用 [typo css](https://typo.sofi.sh/) 进行文章正文的排版。　

### 文章优化

扩展了文章元数据，增加了
- 相关文章
- 外部链接，可以申明本文的转载来源

### seo 优化

扩展了页面和文章的 seo 元数据，支持未每往篇文章或页面设置 seo 信息
- seo 描述　
- seo 关键词　
- seo title

### 深色模式

暂未之前，后续再说啦

## 主题下载

从 github 下载[《snow white》](https://github.com/cmzz/hexo-theme-snow-white)主题

## 参考

在制作主题的过程中，参考了以下几款优秀主题：
- https://d2fan.com/
- http://niexiaotao.cn/
- https://typo.sofi.sh/