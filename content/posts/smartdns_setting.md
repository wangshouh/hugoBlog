---
title: Openwrt系列教程：使用SmartDNS加速DNS解析
date: 2022-02-15T13:53:00Z
tags: [Network,OpenWrt,SmartDNS]
aliases: ["/2022/02/15/smartdns_setting/"]
---

## SmartDNS介绍

SmartDNS是一个运行在本地的DNS服务器，SmartDNS接受本地客户端的DNS查询请求，从多个上游DNS服务器获取DNS查询结果，并将访问速度最快的结果返回给客户端，提高网络访问速度。 同时支持指定特定域名IP地址，并高性匹配，达到过滤广告的效果。

> 详情可见[SmartDNS官网](https://github.com/pymumu/smartdns)

本文主要实现加速访问的效果，暂不考虑其他功能。

## 安装SmartDNS

SmartDNS安装主要参考上述[官网](https://github.com/pymumu/smartdns)，本文将以[OpenWrt luci](https://github.com/openwrt/luci)为例，安装步骤如下：

1. 点击`系统`下的`软件包`选项
1. 在`过滤器`中输入`smartdns`，点击`查找软件包`
1. 下载`luci-app-smartdns`,`luci-i18n-smartdns-zh-cn`,`smartdns`三项

![SmartDNS install](https://img.gopic.xyz/H04ae9198f33a4e1084188571a4e425f8i.png)

> 大部分openwrt编译版本一般都含有此软件包

## SmartDNS的基本配置

见下图：

![Basic SmartDNS Setting](https://img.gopic.xyz/Hf1a79c65b164415a8ec4defb85f3ff241.png)

应注意以下选项：

- `双栈IP优选`，应注意您的设备是否完全支持IPv6，尤其是您所在的网络运营商是否支持

- `重定向`，应选择`重定向53端口到SmartDNS`，但如果您存在其他涉及DNS的软件，应自行选择

## 上游服务器配置

|  DNS服务器名称   | DNS服务器地址 |
|  ----  | ----  |
| 阿里 AliDNS  | 223.5.5.5 |
| 百度 BaiduDNS  | 180.76.76.76 |
| 114 DNS | 114.114.114.114 |
| 腾讯DNS | 119.29.29.29 |
| 山东联通 | 202.102.128.68 |
| Google DNS | 8.8.8.8 |
| CloudFlare | 1.1.1.1 |
| Quad9 | 9.9.9.9 |

最终效果如下图：

![SmartDNS DNS Server](https://img.gopic.xyz/H6e26b18cfcc542db8c3684ba034df4d7t.png)

## 特殊设置

由于SmartDNS具有一般的DNS能力，我们可以在`域名地址`中进行指定的域名解析IP。

```
address /home.lan/192.168.10.50
```

如上述设置的作用是，当访问`http://home.lan`时，将解析成`192.168.10.50`，即我个人的路由器地址。

## 客户端设置

在Windows 10中，通过以下步骤完成DNS设置：

1. 点击`Windows 设置`下的`网络和Internet`选项

2. 点击`状态`中的`属性`，如下图：

![Windows DNS Setting](https://img.gopic.xyz/H56f0a27be9fc4e3695bb8a5ef55975bfE.png)

3. 下滑到`IP设置`，点击`编辑`，如下图：

![IP setting](https://img.gopic.xyz/H8590293925134362bee4497f53e5037dN.png)

4. `IP地址`填入任一未被占用的IP地址，`子网前缀`填入`32`,网关填入`192.168.10.50`，DNS服务器填入`192.168.10.50`，点击`保存`

## 测试

测试方法1：

打开浏览器，输入`http://home.lan`，如果生效可以直接访问到软路由主页

测试方法2：

打开`cmd`，输入`nslookup baidu.com`，如果设置正确则结果如下图：

![Test](https://img.gopic.xyz/H3e9b0317741d4d85a585e710ac9f4c80V.png)

> 配置正确返回的IP地址仅有一个，若返回多个则配置不正确