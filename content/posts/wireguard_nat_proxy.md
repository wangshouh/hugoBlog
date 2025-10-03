---
title: 基于wireguard与NAT实现windows10代理服务器
date: 2022-02-11T09:50:30Z
tags: [WireGuard,Network]
aliases: ["/2022/02/11/wireguard_nat_proxy/"]
---

### 前言
WireGuard是一个免费的网络通信隧道，它可以让您的私有网络和公共网络之间通过一个隧道，让您的私有网络不会被窃听。一般情况下，我们选择使用Linux服务器自带的iptables实现路由转发，实现私有网络和公共网络之间的通信。但是，在Windows上，尤其是个人Windows计算机上，我们缺乏有效的工具实现路由转发，因此我们很难实现私有网络和公共网络之间的通信。搭建出的WireGuard隧道毫无意义。此教程主要讲解如何使用Windows的NAT实现路由转发，并进一步实现私有网络和公共网络之间的通信。

### 环境搭建

### 安装WireGuard

由于本文主要涉及到Windows的NAT实现路由转发，所以我们仅介绍Windows下的安装方法。此过程较为简单，可自行可以参考[WireGuard安装](https://www.wireguard.com/install/)中的介绍。

### 生成WireGuard服务器端配置文件

在后文描述中，我们假定计算机A为服务端，计算机B为客户端。我们需要生成服务端的配置文件，以便计算机B能够连接到服务端后访问公共网络。

我们需要生成服务端的公钥和私钥。打开WireGuard软件，使用 `Ctrl+N` 快捷键创建新的隧道。

将隧道名称设置为 `wg0` ，将公钥复制保存到本地。

然后，输入以下配置内容


```
ListenPort = 55555
Address = 10.254.0.1/24
```

其中，`ListenPort`为隧道监听端口，`Address`为隧道的IP地址和子网掩码。此处选择端口`55555`进行监听，使用了`10.254.0.1/24`的IP地址作为内网IP地址。

> /24表示子网掩码为24，即255.255.255.0

最终效果如下图所示：

![服务端基础配置](https://img.gopic.xyz/H9901cef7da9a40969c94a254b8f696e0N.png)

### 生成WireGuard客户端配置文件

在计算机B上，我们需要生成客户端的配置文件。与上述服务端配置文件相同，我们需要生成客户端的公钥和私钥，并保存公钥钥到本地。该隧道命名为 `wg1` 。

然后，输入以下配置内容

```
Address = 10.254.0.2/32
```
其中，`Address`为客户端的IP地址和子网掩码。

>注意此处设置IP地址需要与服务器端的IP处于同一网段。

![客户端基础配置](https://img.gopic.xyz/H52c11e9f3d1f410696126c0c640ef371V.png)


### 完善WireGuard服务端配置文件

编辑服务端配置文件 `wg1` ，键入以下内容

```
[Peer]
PublicKey = w1bok71zGcFWSAmWCZIE3+ZijCZbrnpzJwrNm7RXqFg=
AllowedIPs = 10.254.0.2/32
```

注意替换`PublicKey`为计算机B的公钥，`AllowedIPs`为计算机B的IP地址。

>`AllowIPs`为路由的IP地址段，可以设置多个，用逗号分隔。其含义为若服务器接收到此IP地址的数据包，则转发给私有网络。通常是该 Peer 的隧道 IP 地址和通过隧道的路由网络。


### 完善WireGuard客户端配置文件

编辑客户端配置文件 `wg1` ，键入以下内容

```
[Peer]
PublicKey = QwiiphIK/UBvbV+n8L4/VOmBlLwSqXt2g0PoY0oJBhY=
AllowedIPs = 0.0.0.0/0
Endpoint = 
PersistentKeepalive = 25
```

注意替换`PublicKey`为计算机A的公钥，`Endpoint`为计算机A的IP地址，`PersistentKeepalive`为保持连接的时间，单位为秒。

>`AllowedIPs`设置为`0.0.0.0/0`的含义为路由此计算机的所有流量至私有网络。

> `PersistentKeepalive`该参数可不设置，但设置了该参数，则客户端会在每次连接后发送心跳包，保持连接。

### 测试连接

打开`WireGuard`客户端，选定`wg2`为连接的隧道，再点击`连接`按钮。

如果连接成功，可以看到对端以及上次握手时间。

如下图：

![连接测试](https://img.gopic.xyz/Hd1f4859a4ebe41b799cf42a44a089ca45.png)


### 设置服务端NAT转发

即使测试连接成功，我们发现客户端仍无法连接入公共网络。原因在于Windows服务端没有流量转发设置。在Liunx上，我们可以通过`iptable`等设置流量转发，但在Windows 10上我们缺乏响应的工具。经过测试，我们发现，如果我们设置了NAT转发，则客户端可以连接公共网络。

以下主要内容主要来自[微软](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network)，如果您的英语水平较高，可以自行查阅。

主要分为以下几个步骤：

1. 以 **管理员** 身份打开`PowerShell`

1. 首先找到需要设置NAT转发的网络接口，键入以下命令`Get-NetAdapter`，可以找到所有的网络接口，并记录`wg0`的`ifIndex`。如下图：
![get-NetAdapter](https://img.gopic.xyz/c52a8b220030702a609ec2f8eebfc4f5.png)

1. 配置NAT网关IP，我们一般使用命令`New-NetIPAddress`,完整命令如下：
```PowerShell
New-NetIPAddress -IPAddress <NAT Gateway IP> -PrefixLength <NAT Subnet Prefix Length> -InterfaceIndex <ifIndex>
```

> `<NAT Gateway IP>`替换为NAT网关IP地址，在上述操作中为`10.254.0.1`

> `<NAT Subnet Prefix Length>`为NAT网关的子网掩码长度，在上述操作中为`24`

>`<ifIndex>`为`wg0`的`ifIndex`

4. 创建NAT网关，使用命令：

```PowerShell
New-NetNat -Name <NATOutsideName> -InternalIPInterfaceAddressPrefix <NAT subnet prefix>
```

> `<NATOutsideName>`为NAT网关的名称，可随意命名

> `<NAT subnet prefix>`为NAT网关的子网掩码长度，在上述操作中为`10.254.0.0/24`

完整命令执行图如下：

![NAT](https://img.gopic.xyz/3744684756d7dd0845197947b611ac28.png)

### 测试

使用客户端连接服务端，访问任一外部网络资源，如可成功访问则证明NAT转发成功。

以安卓客户端为例，展示效果：
使用 Wireguard 前，网络 IP 为中国移动，如下图

![Before](https://img.gopic.xyz/NotWithWireguard.jpeg)

使用 Wireguard 链接 windows 客户端后，网络 IP 显示为 windows 的联通 IP ，如下图

![After](https://img.gopic.xyz/WithWireguard.jpeg)
