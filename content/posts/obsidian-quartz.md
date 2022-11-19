---
title: "使用Quartz部署Obsidian笔记网站"
date: 2022-10-06T11:47:33Z
tags: [hugo,obsidian]
aliases: ["/2022/10/07/obsidian-quartz"]
---
## 概述

使用双链笔记[Obsidian](https://obsidian.md/)进行个人知识管理日益成为主流。但部署`Obsidian`展示网站属于付费功能，这对于很多人来说是一个严重的阻碍。本文将介绍如何使用开源且免费的[Quartz](https://github.com/jackyzha0/quartz)进行部署`Obsidian`笔记并介绍如何进行持续集成和部署。

在继续阅读前，读者可以前往[此网站](https://quartz.jzhao.xyz/)观察部署后的结果。

## 前置条件

阅读本文需要读者具有以下条件:

1. 拥有一个Github账户
1. 熟悉简单的Git概念，如`stage`和`commit`等

## Fork仓库

首先前往[Quartz](https://github.com/jackyzha0/quartz)的仓库地址，点击页面中的`Use this template`按钮，如下图:

![Fork](https://img-blog.csdnimg.cn/img_convert/cf7a0a2888e7b093056d83742c0e82dc.png)

点击后会获得如下页面:

![Fork Result](https://img-blog.csdnimg.cn/img_convert/6d9b477330fb4572a6b2635af5a97666.png)

自行设置仓库名`Repository name`和仓库描述`Description`字段，设置完成后点击`Create repository from template`按钮。

## 设置个人域名

如果读者拥有个人域名可进行这一步。当然，也可以使用`github`分配的`<用户名>.github.io`的二级域名。

前往个人`Fork`后的仓库，点击`Setting`，然后点击`Pages`选项卡，如下图:

![Pages Setting](https://img-blog.csdnimg.cn/img_convert/a1b8f362e7ae1a7d90fe783b2b4f01bc.png)

在输入域名前，读者需要前往个人域名的`DNS`设置一项`CNAME`记录，记录需指向`<用户名>.github.io`，如我的`Github`用户名为`wangshouh`，则需要设置域名指向`wangshouh.github.io`。如果使用[CloudFlare](https://www.cloudflare.com/)管理域名，读者可通过下图步骤增加解析:

![Add CNAME](https://img-blog.csdnimg.cn/img_convert/b57da1ddd9ad051988ab0ad199db088f.png)

在`Custom domain`内填写域名，点击`Save`即可，如果出现错误，`github`会给出提示。

## 设置自动部署

继续在`Pages`页面内进行设置，我们需要将`Branch`内的内容调整的与我相同，即下图红框内的内容应与图片保持一致:

![Branch Setting](https://img-blog.csdnimg.cn/img_convert/a7ded228e417afd8529152bea03afe30.png)

然后，我们需要前往`Code`选项卡，点击`.github`文件夹，如下图:

![Github Code](https://img-blog.csdnimg.cn/img_convert/af12d9a92175cc0440f93f8655533bbe.png)

然后，继续点击内部的`workflows`文件夹，再点击`deploy.yaml`文件，点击下图内红框内容:

![Edit](https://img-blog.csdnimg.cn/img_convert/d51adf3cbf07740406e31a2233cfe347.png)

将文件最后部分中的`canme`更改为自己的域名。如果读者没有个人域名，请更改为`<用户名>.github.io`的格式。修改后，代码如下:
```yaml
- name: Deploy
uses: peaceiris/actions-gh-pages@v3
with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./public
    publish_branch: master  # deploying branch
    cname: 个人域名
```

修改完成后，点击右上角的`Start commit`按钮。

当然，我们还需要修改`config.toml`中的配置内容，如下图:

![Config Toml](https://img-blog.csdnimg.cn/img_convert/627b4c69ff3932354c17045cc1097b79.png)

点击修改按钮，将`baseURL = `后的域名更改为个人域名或`<用户名>.github.io`的形式。

当然，如果读者使用汉语写作，我建议读者将`languageCode`设置为`zh-cn`。如果读者有`googleAnalytics`可自行设置相关字段，如果不明白`googleAnalytics`可直接删除此行。

最后，读者可以前往`Actions`选项卡内观察部署是否完成，如果部署完成，读者可以前往自己设置或`<用户名>.github.io`，应该可以获得如下显示:

![Page Review](https://img-blog.csdnimg.cn/img_convert/b6d462d366b43c98b8c1ee219edac4e3.png)

## Clone 和编辑

我们需要将远程仓库`Clone`到本地已完成笔记编写等任务。读者可以使用任何自己熟悉的`git`工具。我使用的是[sublime merge](https://www.sublimemerge.com/)作为`git`的可视化客户端。

在`Github`网页内获得仓库地址，如下图:

![Github Clone](https://img-blog.csdnimg.cn/img_convert/066d4db06b8e9f469e9eb900040c9bc9.png)

将对应的信息填写到`sublime merge`中即可:

![Sublime Merge](https://img-blog.csdnimg.cn/img_convert/e86947e5f941ac001adc43bd76bdcb6c.png)

其中，`source URL`就是我们在`github`中获得的仓库地址。

`Clone`到本地后，我们可以先修改网站的部分设置，打开`data`文件夹，使用任一文本编辑器打开`config.yaml`文件，读者可以自行修改内部的信息。关于每个字段的具体含义，可自行参考[Config](https://quartz.jzhao.xyz/notes/config/)页面。

一般来说，我们需要修改`name`、`GitHubLink`、`page_title`、`links`字段，依次代表名称、Github仓库的`content`地址、网页标题、社交媒体链接。

修改完成后点击保存，我们可以在`Sublime Merge`内看到以下内容:

![Fix Config](https://img-blog.csdnimg.cn/img_convert/39a3f675509faea41f5c1ff1177c4ede.png)

点击`stage`，并在最上方的`Commit Message`中填入相关信息，点击`commit`即可。点击右侧的`↑`图标，即可完成推送`commit`。

![Push](https://img-blog.csdnimg.cn/img_convert/819279ff3b034e96ccbb5950a1e4d34e.png)

> 由于每次`push`都会触发`github`的部署操作，所以不建议大家进行一次`commit`后直接`push`

## Obsidian 设置

打开`Obsidian`软件，如果第一次打开，会得到如下显示:

![Obsidian new](https://img-blog.csdnimg.cn/img_convert/0c310bb939bc1a093401a608a6254b6e.png)

选择`打开本地仓库`，选择地址为`clone`仓库文件夹下的`content`文件夹。

打开`Obsidian`设置菜单，设置以下参数:

![Obsidian Setting](https://img-blog.csdnimg.cn/img_convert/430aeac1bc97ee7e96ea498b2618c46b.png)

完成此步骤后，基本就已经完成了全部设置。读者可以自由地编写笔记，当然，我们也需要使用`Sublime Merge`在完成笔记后进行`commit`和`push`操作。

![Megre Note](https://img-blog.csdnimg.cn/img_convert/312bed16e3c1125a9c5788b618d912a0.png)

> 注意，不要删除`content`下的`template`文件夹，且不要使用`index.md`作为文件名

更多复杂设置，读者可以自行参考[Setup](https://quartz.jzhao.xyz/tags/setup/)网页。
