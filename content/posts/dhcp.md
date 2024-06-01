+++ 
draft = false
date = 2024-06-01T23:07:52+08:00
title = "DHCP 协议"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

新开了一个支线任务：探索Linux Common Port
- 本次任务： 

| 端口       | 描述                                                             |
| -------- | -------------------------------------------------------------- |
| 67 (UDP) | Used by the DHCP server (Dynamic Host Configuration Protocol). |
| 68 (UDP) | Used by a DHCP client.                                         |


67和68两个端口用于DHCP协议，分别是服务端和客户端  
DHCP是应用层协议，传输层协议使用的是UDP


## 什么是DHCP
DHCP(Dynamic Host Configurration Protocol)协议，动态路由配置协议，用于动态分配IP


你走进咖啡厅，用手机连接咖啡厅的公共wifi，此时你的手机将需要被动态分配一个IP，才能用语网络通信，那么这个动态IP由谁负责给你，又是通过什么方式给你呢？

一般负责动态IP分配的是路由器，而分配动态IP所遵循的协议就是DHCP，下面就是你从连接咖啡厅wifi到分配ip的过程：

1. 当你的手机连接到咖啡厅的公共WiFi网络时,它会向WiFi路由器发送请求,希望获得一个IP地址以接入互联网。
2. 路由器上运行着DHCP(动态主机配置协议)服务,它负责动态分配IP地址。DHCP服务器会从可用的内网IP地址池中选择一个空闲的IP地址,并将其分配给你的手机。
3. DHCP服务器把分配的IP地址、子网掩码、默认网关等信息以DHCP协议的方式发送回你的手机。
4. 你的手机收到DHCP服务器的响应后,就可以使用这个动态分配的内网IP地址来接入互联网了。

上面获取动态IP的过程就是DHCP

## DHCP 的具体过程
简单来说，四次通信即可完成。

(Discover) client -> servers 广播 Hi，当前广播区域内谁可以给我分配一个动态IP？  
(Offer) server A -> client 单播 我可以给你一个动态IP地址！你的地址是xxx，子网掩码是xxx，dns服务器是xxx，以及其他有用的信息  
(Request) client -> servers 广播 我觉得server A 很不错，我要选定server A提供给我的动态IP地址  
(ACK) server A -> client 单播 这里是你的动态IP地址！你的地址是xxx，子网掩码是xxx，dns服务器是xxx，以及其他有用的信息  

## 为什么DHCP需要四次通信

可以支持多个DHCP服务器的情况，当设备进入陌生网络，并不知道其中有几个DHCP服务器，只能**广播**发送一个Discover来表明需求，这时候可能收到多个DHCP服务器的OFFER，需要从中选定一个，还是**广播**形式，同时也能告知其他DHCP服务器你选择了其中哪一个OFFER


就像毕业时你在论坛上吐槽说：”老天赐我一个OFFER吧“，然后Google，Meta，X等多家企业纷纷像你投来橄榄枝，你纠结一番决定去Google，然后在论坛上告诉各大公司，你们的OFFER婉拒了哈，去Google了！（纯属想🍑。。。）


## 为什么有时DHCP只要两次通信
当我用wireshark抓包，关闭并重新连接电脑的wifi时，发现只有后两个步骤，也就是request和ack，可能是因为电脑已经知道DHCP server的地址，所以discover请求被省略

简单来说就是客户端已经有了一些DHCP服务的上下文，不需要再重新广播去Discover一个DHCP服务器

![](https://pic-1258720617.cos.ap-beijing.myqcloud.com/202405290004184.png)
