---
author: Huang Yanchen
title: 记一次Vs Code ssh remote连接失败
date: 2024-08-19
description: VS Code ssh remote failed to start
math: true
ShowBreadCrumbs: false
tags: ["VS Code", "ssh remote"]
categories: ["Coding"]
showToc: false
TocOpen: false
UseHugoToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
---
事情是这样的，我平日里一直会使用 vscode 连接学校平台提供的远程服务器进行开发，在使用 ssh remote 时输入要连接的 ip 地址加端口号，配置文件就会自动保存在 username/.ssh 文件夹下的 config 中。

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408191707623.png)

学校平台内每次申请的可以交互的开发环境是存在时间限制的，在时间到期之后申请的服务器资源就会自动释放，之后如果再申请其他的资源，放给你连接的 ip+port 是没有变化的，所以看起来可以直接连接，但是在多次尝试后，都得到了报错信息:

```shell
Host key verification failed. 过程视图写入的管道不存在
```

在辗转了一些网页后，最终在这篇 [CSDN 文章](https://blog.csdn.net/dive668/article/details/126103503)中找到了想要的结果。

只需要删除 known_hosts、known_hosts.old 文件中，对应出问题的 ip 行，然后重新使用 ssh remote 进行连接即可。
