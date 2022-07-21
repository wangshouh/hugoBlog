---
title: 以Openwrt路由器为核心的家庭网络构建
date: 2022-02-1T13:53:00Z
tags: [Network,OpenWrt]
aliases: ["/2022/02/14/home_network/"]
---

## 环境概览

家庭的基本情况如下：
- 三层楼房，总建筑面积较大
- 千兆入口光纤
- 主要活动范围为一楼

家庭的主要硬件如下：
- 软路由器，系统为openwrt
- 千兆双频无线路由器
- 百兆无线路由器，品牌为TP-Link
- 联通自带千兆光猫
- 多部手机
- 具备千兆网卡的笔记本电脑

## 网络接线情况

![Wiring Diagrams](https://s-bj-3358-blog.oss.dogecdn.com/network.svg)

联通光猫与接入光纤直接相连，光猫已设置为桥接模式并不进行拨号，功能仅为转换光电信号。

光猫任选一个Lan口，与后端千兆无线路由器的Wan口连接。由于此网线承载这个家庭网络，故应尽可能选择高规格网线，此处我个人选择了六类网线。此处千兆无线路由器主要负责一楼的无线网络。

> 网线规格表
![Wireless Cable Specification](https://ae01.alicdn.com/kf/H9fc2a5ff43a4420fb4535275cc0ec828e.png)
> 该表来源于[Wikipedia](https://zh.wikipedia.org/wiki/%E5%8F%8C%E7%BB%9E%E7%BA%BF)

千兆路由器任选Lan口与斐讯N1软路由连接，再选一个Lan口与二楼的百兆路由器的Lan口使用五类网线相连接。该百兆路由器使用2.4GHz频段，负责二楼与三楼的无线网络。

## 路由器配置

### 主无线路由器配置（千兆路由器）

Wan口设置为PPPoE拨号，PPPoE拨号用户名和密码自行设置。

Lan口IP设置为 `192.168.10.1`，子网掩码设置为 `255.255.255.0`。
DHCP服务器开启，地址池设置为开始地址为 `192.168.1.100`，结束地址为 `192.168.1.255`。

### 副无线路由器配置（百兆路由器）

注意此处为保证家庭网络位于同一网段，我们需要特殊设置，使路由器启用AP交换机模式。具体内容可参考以下[教程](https://service.tp-link.com.cn/detail_article_4145.html)。

简单来说，就是放弃Wan口，仅使用Lan口，并在Lan口设置DHCP服务器。Lan口IP设置为 `192.168.10.2`，子网掩码设置为 `255.255.255.0`。DHCP服务器开启，地址池设置为开始地址为 `192.168.1.3`，结束地址为 `192.168.1.49`，网关为 `192.168.10.1`。

### 软路由配置

打开软路由器的设置界面，点击“网络”，点击“接口”，修改“br-lan”，各项设置如下图：

![Setting](https://ae01.alicdn.com/kf/H743cc3f6c5ff434280b20d5630250861P.png)

> 注意此处DHCP服务器选择`忽略此接口`

## 实现效果

家内所有设备位于同一网段并可互相访问，这对软路由器为全家设备服务是很重要的前提。

## 拓展阅读

- [x] [Openwrt系列教程：使用SmartDNS加速DNS解析](https://wangshouh.github.io/2022/02/15/smartdns_setting/)

- [ ] Openwrt系列教程：使用WireGuard实现VPN访问

- [ ] Openwrt系列教程：使用Aria2实现离线下载

- [ ] Openwrt系列教程：使用smb实现共享文件

- [ ] Openwrt系列教程：使用动态DNS实现动态访问