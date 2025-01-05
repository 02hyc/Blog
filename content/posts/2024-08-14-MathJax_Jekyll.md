---
author: Huang Yanchen
title: 记一次MathJax渲染失败 | MathJax parsed in kramdown in Jekyll
date: 2024-08-14
description: A brief guide to setup KaTeX
math: true
ShowBreadCrumbs: false
tags: ["Jekyll", "MathJax"]
categories: ["Blog"]
showToc: false
TocOpen: false
UseHugoToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
---
上传了前一篇博客后，我像往常一样打开网站，却发现这篇文章中的数学公式都没有在网页上正确的渲染出来，怪异的符号在屏幕上扭曲，让我感到有点难堪，于是我第一时间检查了我 fork 的原仓库中的 issue，里面并没有太多关于公式渲染的内容。

印象中，[fork 的原仓库](https://github.com/mmistakes/so-simple-theme)中，是有一篇 example post 讲解了关于网页内数学公式的渲染问题的，于是查看了那篇[示例说明](https://github.com/mmistakes/so-simple-theme/blob/master/docs/_posts/2015-08-10-mathjax-example.md)，里面只对 config 部分的配置做了说明：

```yaml
markdown: kramdown
mathjax:
  enable: true
  combo: "tex-svg"
  tags: "ams"
```

我兴冲冲地跑去查看我的_config 文件，果不其然，发现 mathjax 这里，并没有打上 `true` 的标签，加上标签之后，我在 wsl2 内本地启动网页，期望能够立马收获成效，然而，这一次依然失败了。

到底是什么原因导致我的公式没有办法被正确渲染呢？我在 google 上搜索关键词，kramadown, jekyll, mathjax，在各个仓库的 issue 和 stackoverflow 中，尝试了很多办法，都没有成效。在[博客](https://zhengyuan-public.github.io/posts/PersonalBlogWithJekyll/#latex-math-equations-in-jekyll)中，我理解了 jekyll 项目中，数学公式的渲染过程：
$Markdown\xrightarrow{\text { kramdown }} HTML \xrightarrow{\text { MathJax }} Math Equation Images$

使用 kramdown 将原生 mardown 中的公式解析至 html 中，然后通过 MathJax 将其解析成一张 svg 图片。如果公式只是图片形式显示的话，那可玩性似乎下降了很多，我也查看了一些其他同学的 blog，他们博客中的公式并不是以图片形式呈现的。对于建站我了解的并不是很多，因此选择使用了一个 Jekyll 项目的模板来搭建，也仅仅是想供我有个执笔之处，或许在未来，对建站方面的知识了解更多后，可以对我的网页进行大的重组。

但这是多久之后的未来......

话说回来，尝试了很多网络上搜索的办法都逐一失败了，失败的记录我也全都丢掉了。于是我转而询问 Tongyi 和 GPT，Tongyi 最初给我的回答和网络上那些方法无异，在这里贴出部分内容：

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408141505219.png)

顺着它的提示，我尝试在公式前后加上 `<script type="math/tex">` 标签，再一次本地启动，发现在这个标签内的公式居然被正确解析出来了。

于是我觉得，是否是 MathJax 中，解析的关键词出了问题呢？我再一次进行了询问，这一次，Tongyi 给我的 MathJax 配置代码和最终成功案例十分接近：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>MathJax Example</title>
  <script type="text/javascript" id="MathJax-script" async
    src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
  </script>
  <script type="text/javascript">
    MathJax = {
      tex: {
        inlineMath: [['$', '$'], ['\\(', '\\)']],
        displayMath: [['$$', '$$'], ['\\[', '\\]']],
        processEscapes: true,
        processEnvironments: true,
        tags: 'ams' // Use AMS (LaTeX standard) tags for numbering
      },
      options: {
        ignoreHtmlClass: '.*|',
        processHtmlClass: 'process-math'
      }
    };
  </script>
</head>
<body>
  <!-- Your Markdown content goes here -->
  <p>Here is an inline formula: $E = mc^2$. And here is a displayed formula:</p>
  <p>$$ E = mc^2 $$</p>
  <p>And another one:</p>
  <p>\[ E = mc^2 \]</p>
</body>
</html>
```

这次的代码中，显式地声明了‘$’作为内联和行间公式的符号，我觉得思路是正确的，但是依然没有得到成功。我查看了网页的控制台内是否是 MathJax 解析报错的信息，可惜也没有看到。

于是我询问 GPT-4o-mini，一开始他给出了我和第二次 Tongyi 相同的答案。我将所有的 script.html 中的内容都输入给 GPT，再次询问，这一次，GPT 列出的几个方案中，有一个方案吸引了我的目光：

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408141505210.png)

我的 MathJax 配置本来是这样的：

```html
MathJax = {
  tex: {
      inlineMath: [['$', '$'], ['\\(', '\\)']], 
      displayMath: [['$$', '$$'], ['\\[', '\\]']],
      tags: "{{ site.mathjax.tags | default: 'ams' }}"    // eq numbering options: none, ams, all
  },
   options: {
     renderActions: {
       // for mathjax 3, handle <script "math/tex"> blocks inserted by kramdown
       find: [10, function (doc) {
         for (const node of document.querySelectorAll('script[type^="math/tex"]')) {
           const display = !!node.type.match(/; *mode=display/);
           const math = new doc.options.MathItem(node.textContent, doc.inputJax[0], display);
           const text = document.createTextNode('');
           node.parentNode.replaceChild(text, node);
           math.start = {node: text, delim: '', n: 0};
           math.end = {node: text, delim: '', n: 0};
           doc.math.push(math);
         }
       }, '']
     }
   }
}
```

我并不太理解 options 的含义，但凭直觉感觉大概意思是对 `<script "math/tex">` 这个标签进行解析，而在之前的尝试中，也仅仅只能解析出这个标签下的内容。我将 options 部分全部删去，不抱希望地在本地启动网站后，发现在页面中数学公式都被正常的渲染出来了，到这一刻才真相大白，options 中的内容似乎和正常的渲染起了冲突。

以下是 GPT 的解释：

> 这表明 `renderActions` 部分的代码可能在你的特定环境中导致了与 MathJax 的解析冲突。虽然 `renderActions` 是用于处理 `<script type="math/tex">` 标签中的数学表达式的，但在你的配置中可能并不需要，或者它与其他配置不兼容。
> 既然去除后公式可以正常渲染，你可以保留简化后的配置，并确保你的 MathJax 配置文件简单且有效。对于处理行内公式（$...$）和行间公式（$...$），目前的配置已经足够了。如果将来需要处理更多特定的场景，再考虑添加其他配置项。

似乎 GPT 的资料中也并没有对这一部分十分明晰。

在最后，我尝试进行了消融实验。鉴于以下部分也是 Tongyi 提示后加入的，我试着删去以下内容，查看这一次页面是否能够正确渲染公式：

```html
inlineMath: [['$', '$'], ['\\(', '\\)']], 
displayMath: [['$$', '$$'], ['\\[', '\\]']]
```

然而在删去了这部分之后，公式似乎不能正确渲染了。显示地声明‘$’符号并不是冗余的过程。国产大模型中，在日常代码使用方面，通义千问确实给我最好的体验，而 GLM4 的体验则十分糟糕，尽管它貌似在多个测试套件上跑分很高。基于通义千问的代码辅助插件通义灵码也早已推出，但由于我已经有了 Copilot，就不花精力去测评灵码了。但我目前依然觉得，代码能力方面，通义系列依然是国产大模型的答案。
