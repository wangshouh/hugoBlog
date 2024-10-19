---
title: 在Cloudflare中自动化部署hexo博客
date: 2022-06-27T10:47:33Z
tags: [cloudflare,hexo]
aliases: ["/2022/06/27/cloudflare-hexo/"]
---

## 概述

`CloudFlare Pages`是一个免费且使用简单的`CI/CD`工具，可用来编译并部署大部分知名前端框架。如果您查阅`cloudflare Pages`的文档会发现其支持`hexo`框架的编译和发布。但如果您直接进入cloudflare pages dashboard进行设置时，你会发现在`Build settings`中的`Framework preset`并没有给出`hexo`框架的选项，如下图。

![clouflarepagesSetting.png](https://acjgpfqbqr.cloudimg.io/_s3_/clouflarepagesSetting.png)

本篇博客将使用比较优雅的方法解决此问题。

## 解决方案

在其他人的解决方案中给出了一种使用github action编译后发布到github pages仓库再使用cloudflare进行直接分发的方法，该方法存在许多问题，主要问题在于你很难进行`hexo`配置文件的编写。在此博客中，我们会采用我个人摸索出的更加优雅的方法。

在cloudflare pages dashboard中，直接点击`Create a project`按钮，选择好你所需要链接的代码仓库或其他内容，我选择了使用github仓库作为演示。选择完成仓库后，点击`Begin setup`。在`Build settings`，各项设置如下表：

| 设置项 | 内容 |
|--|--|
| Framework preset | None |
| Build command | hexo generate |
| Build output directory | public |

设置完以上内容后，直接点击`Save and Deploy`即可完成在cloudflare pages上的部署。如果您选择了和我一样使用github仓库，当你每向你选定的分支进行一次推送后，cloudflare pages将自动进行部署，实现优雅的自动化部署，如下图：

![cloudflareshow.png](https://acjgpfqbqr.cloudimg.io/_s3_/cloudflareshow.png)

## 环境变量设置

由于 CloudFlare 默认使用的`npm`和`nodejs`版本较低，在部署时可能出现如下错误:
```log
ERROR ReferenceError: /opt/buildhome/repo/node_modules/hexo-theme-keep/layout/page.ejs:11
```

如果出现这种错误，我们可以通过设置`Cloudflare`环境变量的方法进行解决，打开部署详情页，在`settings`栏目中的`Environment Variables`选项内增加以下内容:

| Variable name | Value |
| -- | -- |
| NODE_VERSION | 16.17.1 |
| NPM_VERSION | 8.15.0 |

最终结果如图:

![Cloudflare Env](https://acjgpfqbqr.cloudimg.io/_csdnimg_/bd2a6e597e46d685f1c03bd561a56de1.png)
