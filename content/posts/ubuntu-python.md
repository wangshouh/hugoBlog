---
title: "Ubuntu Python 多版本安装"
date: 2023-07-05T11:47:33Z
tags: [python]
---


## 概述

由于 Python 3 有几次较为跳跃的更新，导致大量使用 Python 3 作为开发工具的软件会对 Python 3 的版本进行严格限制，如限制使用 Python 3.8 - Python 3.9 版本。这要求开发者开发环境内应具有多版本的 python 。在 Ubuntu 等 Linux 系统下，Python 的安装都是使用的源码编译方法，这对一些 Python 开发者并不友好，本文会给出一种较为简单的多版本 Python 安装方法。

## 安装

笔者使用的 Ubuntu 22.04 自带有 Python 3.10 版本的 Python ，可以使用以下命令查看系统自带的 Python 版本:

```bash
python3 --version
```

笔者希望安装 Python 3.9 并创建相关虚拟环境。

首先，我们需要引入新的 apt 源，如下:

```bash
add-apt-repository ppa:deadsnakes/ppa
```

[deadsnakes](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa) 在其软件仓库中给出了大量 Python 版本的安装包，这些安装包都是预编译好的，我们不需要进行进一步的编译。

以安装 Python 3.9 为例，命令如下:

```bash
apt install python3.9
```

安装完成后，我们可以使用以下命令查看 Python3.9 是否安装成功:

```bash
python3.9 --version
```

接下来，由于有了多版本 Python ，那么设置虚拟环境是必须的，使用以下命令安装虚拟环境套件:

```bash
apt install python3.9-venv
```

使用以下命令创建虚拟环境:

```bash
python3.9 -m venv test
cd test
```

并使用 `source bin/activate` 命令激活虚拟环境，读者可以在虚拟环境中使用 `pip` 命令进行安装相关 `pip` 包。在虚拟环境激活后，`python3 --version` 的输出也发生了改变，说明虚拟环境中的 Python 3.9 掩盖了外部的 Python 版本，这对于多版本管理是有用的。最后，读者可以使用 `deactivate` 退出虚拟环境。

限于篇幅，我们无法介绍关于 `venv` 的全部情况，请读者自行参考 [文档](https://docs.python.org/3.12/tutorial/venv.html)。

最后，有部分读者可能会安装一些特殊的 `pip` 包，如 `fastecdsa ` 等，这些包都是使用 C 语言编写，安装时需要编译，为解决这些包的问题，我建议读者安装以下库:

```bash
apt install python3.9-dev
```

该库会安装 Python3.9 的开发工具，对于需要编译安装的库来说，该包是必要的。
