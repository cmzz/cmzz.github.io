---
title: Web 前端直接渲染 Office 格式文档的几种方案
tags: 
- 前端
- 技术
date: 2022-05-30 08:00:00
---

在一些中台系统中或管理后台系统中，在线预览 Office 文档是个比较常见的需求，奈何浏览器的支持有限在做相关功能时踩了一些坑。

## 通用解决方案

### 通过 PDF 格式预览

现在注流的浏览器已经支持了 PDF 文件的在线预览，但是对于 word / excel 等格式文件，浏览器的还没有提供直接的支持。应此在需要预览 word 或 excel 文件时，可以考虑先将其转换为 PDF 文件，再通过浏览器的能力渲染。

渲染 PDF 文件非常简单，比较常见的有下面 2 种方式：

- 链接

```html
<p>Open a PDF file <a href="/uploads/media/example.pdf">example</a>.</p>
```

点击链接，浏览器会新一个 Tab 来打开 PDF 文件进行预览。

- 在 html 中通过 iframe 来渲染

```html
<iframe 
    src="/uploads/media/example.pdf" 
    width="100%" 
    height="500px">
</iframe>
```

2 种方式都是基于浏览器内置能力实现，有点和缺点也都比较明显：
- 优点：自带“打印”，“搜索”，“翻页”等功能，强大且实现非常简单方便
- 缺点：不同浏览器的pdf工具样式不一，且无法满足个性化需求，比如：禁止打印，下载等

### PDF.js

[PDF.js](https://mozilla.github.io/pdf.js) 由 mozilla 开发并使用 apache 许可开源发布的工具库。其基于HTML5技术构建，用于展示可移植文档格式的文件(PDF)，它可以在现代浏览器中使用且无需安装任何第三方插件。简单的 demo 如下，详细使用方法可参考项目官网。

```html
<canvas id="pdf-canvas"></canvas>

var url = 'Helloworld.pdf';

PDFJS.getDocument(url).then((pdf) => {
    return pdf.getPage(1);
}).then((page) => {
    // 设置展示比例
    var scale = 1.5;
    // 获取pdf尺寸
    var viewport = page.getViewport(scale);
    // 获取需要渲染的元素
    var canvas = document.getElementById('pdf-canvas');
    var context = canvas.getContext('2d');
    canvas.height = viewport.height;
    canvas.width = viewport.width;
    
    var renderContext = {
        canvasContext: context,
        viewport: viewport
    };
    
    page.render(renderContext);
});
```

### 通过在线预览服务

微软和 Google 都提供了免费的文档在线预览服务，通过该服务我们可以非常方便的实便在 web 中预览文件，但是这种方式的缺点在于，文档必需为公网可访问，否则无法预览。

```html

<!-- 微软 -->
<iframe 
    src='https://view.officeapps.live.com/op/view.aspx?src=http://a.com/b.xls' 
    width='100%' height='100%' frameborder='1'>
</iframe>

<!-- Google -->
<iframe :src="'https://docs.google.com/viewer?url="fileurl"></iframe>
```

除了在公网使用微软服务，或将 word 或 excel 文件转为 PDF 外，还有没有其它的方式呢？

## sheet.js

[SheetJS](https://github.com/SheetJS/sheetjs) 和 pdf.js 类似，基于现在流行的 HTML5 技术构建，可以直接在	web 页面中通过 js 或 ts 渲染表格。

## word 文件

目前对于 word 文件还是无能为力。

## 商业服务

在企业市场，将文档转为共网可访问的文件，显然是不行的，因此催生了众多的商业服务为企业提供相关的解决方案。这里列举几个常见的平台：

- XDOC文档预览云服务 https://view.xdocin.com/    支持私有化
- 永中 https://www.yozodcs.com/   支持私有化
- IDOC https://www.idocv.com/docs.html  支持私有化
- 文档服务 DOC https://cloud.baidu.com/product/doc.html?track=cp:nsem|pf:pc|pp:doc|pu:long|ci:|kw:118945
- Wps https://wwo.wps.cn/docs/introduce/

总结一下，对于可公开的文档，基于微软的在线预览服务，简单便捷。对于不可公开的文档，可以考虑将其转换为通用的 PDF 格式。如果有更多的要求，可以购买相关的服务。