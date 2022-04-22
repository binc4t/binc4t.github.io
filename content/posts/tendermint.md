+++ 
draft = false
date = 2022-04-15T00:41:31+08:00
title = "Tendermint笔记"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++
# 参考资料
[什么是Tendermint](https://www.notion.so/Tendermint-Core-3be8256293ae4c36810d2155477bf34c#d931765e36f7423f877333244aa81d0c) 

# 什么是Tendermint
Tendermint 是一种区块链的构建工具，或者区块链的开发框架，能够帮助开发者快速构建一条区块链。它主要包含两个部分：Tendermint Core 以及 ABCI(Application Blockchain Interface)，前者通过BFT实现了一种共识协议以及P2P网络通信，后者则定义了搭建在 Tendermint Core 之上的应用层接口。

Tendermint Core 将区块链共识协议层和对等网络通信层实现，使得开发者能够不用过分关心复杂的底层实现，而专注于应用层开发。借用[Dan Boneh](https://www.youtube.com/watch?v=V0JdeRzVndI&t=113s)提到的区块链网络分层模型，我认为Tendermint Core实现了下两层的内容，如下图所示。

![区块链网络分层模型](https://yyypics.oss-cn-beijing.aliyuncs.com/Untitled-2022-04-16-2352.svg)

