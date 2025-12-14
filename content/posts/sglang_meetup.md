+++ 
draft = false
date = 2025-12-14T19:56:22+08:00
title = "百度智能云 SGLang Meetup 记录"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

# 百度智能云 SGLang Meetup 记录

今天去百了度组织的 [SGLang Meetup](https://mp.weixin.qq.com/s/BntG0ggzEdUR0ZL_aMq0Ww)，看看明星项目都在解决什么问题，在场好多大佬... 

全场的目标就是集中在GPU利用率以及推理加速上，勉强全部听下来了，很多名词还需要慢慢消化。记录一下印象比较深的几点：

## SGLang 的最新进展
首先是 SGLang 的最新进展， SGLang 支持了 **Diffusion**，从文本大模型的优化推广到对于多模态的优化，尤其是图片生成。  
<div style="text-align: center;">
<img src="https://pic-1258720617.cos.ap-beijing.myqcloud.com/202512142025506.png" width="80%">
</div>

还有另一块提到比较多的方向就是**投机推理**，用小而快的草稿模型来预测多个 Token，再由大模型批量验证，达到推理提速的目的。SGLang 提供的 
 SpecForge 就专门应用于这种场景。

## 推理集群部署
在部署方面，目前优化的方向还是着眼于**DP分离**架构下的GPU利用率优化，例如用batch调度来做整体均衡调度，减少并行气泡；另外我比较感兴趣的是Prefill和Decode之间的**KV Cache通信**问题，也许之前的知识能派上用场。
<div style="text-align: center;">
<img src="https://pic-1258720617.cos.ap-beijing.myqcloud.com/IMG_1720%202.jpg" width="80%">
</div>

## DeepSeek V3 系列
模型适配上面，**DeepSeek V3.2** 模型受到格外的青睐，厂商在对其进行针对性的优化


## 喜闻乐见的茶点环节  
<div style="text-align: center;">
<img src="https://pic-1258720617.cos.ap-beijing.myqcloud.com/IMG_1726.JPG" width="80%">
</div>


## 周边
最后不得不说 SGLang 还是太懂周边，我被这个帆布袋子狠狠拿捏了  
<div style="text-align: center;">
<img src="https://pic-1258720617.cos.ap-beijing.myqcloud.com/IMG_1729%20(1).jpg" width="80%">
</div>