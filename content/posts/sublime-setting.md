---
title: "一个简单的Sublime设置"
date: 2023-03-05T23:47:33Z
---

## 问题

如果读者熟悉我，应该会发现我经常使用 VSCode 作为主力编辑器，但随着我安装的 VSCode 的插件逐渐增加，我发现对于部分较小的任务使用 VSCode 过于笨重，比如简单的 Markdown 文件编辑工作。

在经过一系列寻找后，我发现高性能且占内存较低的 [Sublime Text](https://www.sublimetext.com/) 是符合我的需求的。该编辑器具有大量的语言高亮支持且几乎是开箱即用的。

但该编辑器的部分默认设置是不符合程序员需求的，本文主要解决这些问题。

## 字体

Sublime 的原生字体较为奇怪，对于汉语的支持非常差，但对代码的支持还算可以。

![Sublime Deafult Font](https://acjgpfqbqr.cloudimg.io/_img1_/8447f7102510a1afbbd219e350ffef76.png)

与 VSCode 不同， Sublime 不支持所谓的回退字体，我们需要指定一个包含汉语和英语的对程序员友好的字体。大部分网上的教程都建议使用混合字体，即使用微软雅黑与某种等宽字体混合。但事实上，随着国内开源字体的发展，已经出现了原生的同时支持汉语与等宽英语的字体，本文推荐使用 [霞鹜文楷等宽](https://github.com/lxgw/LxgwWenKai)。此字体在显示中文内容和英文代码时都较为优秀，如下：

![LxgwWenKai Mono](https://acjgpfqbqr.cloudimg.io/_img1_/0c40cddaa7c81d3e766d5f7dfab32976.png)

读者可在 [LxgwWenKai Github Release](https://github.com/lxgw/LxgwWenKai/releases) 内进行字体下载，请注意需下载文件名包含 `Mono` 的字体文件，如下图：

![LxgwWenKai Download](https://acjgpfqbqr.cloudimg.io/_img1_/fdb635306a136c99b3e54466d5feabd9.png)

下载后直接点击安装即可。如下图:

![Font Install](https://acjgpfqbqr.cloudimg.io/_img1_/25757b31a24dd0d22220f8d5a8fa9289.png)

安装后，读者可以在系统字体设置中搜索到此字体:

![Windowns Font](https://acjgpfqbqr.cloudimg.io/_img1_/7ad2b738cedca60ab6cecd796c525a2e.png)

然后我们需要对 `Sublime` 进行简单的设置，打开 Sublime 后点击 `Preferences` 中的 `Setting` 选项:

![Sublime Setting](https://acjgpfqbqr.cloudimg.io/_img1_/b7caafff3b91ea0267fd266c68653246.png)

在配置文件内写入 `"font_face": "霞鹜文楷等宽"` 配置项保存即可。 我的配置文件如下:

```json
{
	"theme": "Default Dark.sublime-theme",
	"font_face": "霞鹜文楷等宽",
	// "font_face": "JetBrains Mono",
	"font_size": 14,
}
```

其中，`theme` 配置编辑器主题，而 `font_face` 配置字体，`font_size` 配置字体大小。

此处读者可以注意到被注释的 `JetBrains Mono` 字体选项，当我需要阅读大量英文代码时，我会选择此字体。读者可前往 [JetBrains 官网](https://www.jetbrains.com/lp/mono/) 获得此字体。

> 切换字体时，我们需要注释掉 `"font_face": "霞鹜文楷等宽"` 然后去掉 `"font_face": "JetBrains Mono"` 的注释，注意 Sublime 仅支持单一字体而不支持字体列表这种配置。(VSCode 允许多字体配置)

## Git 支持

与 VSCode 不同，Sublime 原生仅支持 Git 状态显示而不支持进行 `Commit` 等操作，为达成这一目的，我们需要安装 Sublime 的另一款软件 [Sublime Merge](https://www.sublimemerge.com/)。我们在此不详细介绍安装流程，推荐读者使用 `portable version` 版本下载压缩包，然后将其直接解压。

![Sublime Merge Portable](https://acjgpfqbqr.cloudimg.io/_img1_/947a28ce3be311fdf522e939fbc28cee.png)

读者需要记住压缩包解压后的位置，尤其是解压得到的 `sublime_merge.exe` 的位置，然后在配置项更改如下:

```json
{
	"theme": "Default Dark.sublime-theme",
	"sublime_merge_path": "D:/sublime_merge/sublime_merge.exe",
	"font_face": "霞鹜文楷等宽",
	// "font_face": "JetBrains Mono",
	"font_size": 14,
}

```

读者需要更改其中的 `sublime_merge_path` 地址。

更改完成后，读者可以通过点击 `Sublime Text` 文本编辑器右下角的 `main` 直接唤醒 `Sublime Merge` 进行 Git 操作。

![Sublime Merge](https://acjgpfqbqr.cloudimg.io/_img1_/07ad27d752e5658883a4f5df2567dd6c.gif)

## 插件

又到了喜闻乐见的插件环节，对于 Sublime Text 而言，我个人并没有安装大量插件，对此也并不是非常熟悉。读者可以前往 [插件中心](https://packagecontrol.io/) 寻找喜欢的插件，然后点击 `Perferences` 下的 `packeage control` 打开插件命令面板，从中搜索您需要的插件直接下载即可。

![Install Package](https://acjgpfqbqr.cloudimg.io/_img1_/a80a71f70e1d24be170d926abb027c15.png)

> 注意首次打开此功能时需要缓存一定数据请耐心等待。

对我而言， `Sublime Text` 是一个比较纯粹的文本编辑器，所以我没有过多的安装插件。我个人仅安装了两个插件，如下:

1. [A File Icon](https://packagecontrol.io/packages/A%20File%20Icon) 支持过多文件的图表显示
1. [BracketHighlighter](https://packagecontrol.io/packages/BracketHighlighter) 支持多重括号显示
1. [MarkdownEditing](https://packagecontrol.io/packages/MarkdownEditing) 增加了一些 Markdwon 的功能

我在 Sublime Text 上较为节制，不希望这一简洁的文本编辑器变成 IDE
