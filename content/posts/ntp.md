+++ 
draft = false
date = 2024-06-07T21:59:26+08:00
title = "NTP 协议"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

| **Port**  | **Description**              |
| --------- | ---------------------------- |
| 123 (UDP) | NTP (Network Time Protocol). |

## 什么是NTP协议
NTP(Network Time Protocol) 网络时间协议，运行在123端口上， 用于时间同步


NTP 通过分级的方式来组织所有节点，从0到15，共15级，0级是最准确的时间设备，从0级同步时间的设备是1级设备，依次类推

网络上的设备是一个既做服务器又做客户端的模型，在从其他设备获取时间后，其他设备同样可以从你这里获取时间

具体linux设备上，可以启动ntpd，ntpd通过ntp协议来同步时钟

## NTP协议的原理
NTP原理简单理解起来就是，
假设A想从B初同步时间，那么需要向B发起请求，B把自己的时间发给A，但是因为网络延迟的原因，B的消息传达给A要过一个单程延迟，因此A需要知道单程延迟是多少

从A和B的两次通信中，其实就可以估算出网络延迟

借用[此处](https://info.support.huawei.com/info-finder/encyclopedia/zh/NTP.html)的图片来说明  

![](https://pic-1258720617.cos.ap-beijing.myqcloud.com/202406072215855.png)

其中   
t1 是客户端发送数据时, 客户端的时间
t2 是服务端收到数据时, 服务器的时间
t3 是服务端发送数据时, 服务器的时间
t4 是客户端收到数据时, 客户端的时间


利用这四个时间就可以估算出延迟offset  
![](https://pic-1258720617.cos.ap-beijing.myqcloud.com/202406072215103.png)

client就可以用t3 + offset 来校准时间
