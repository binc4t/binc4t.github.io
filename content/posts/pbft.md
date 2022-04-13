+++ 
draft = false
date = 2022-04-12T23:36:57+08:00
title = "pbft笔记"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

# 参考资料
[PBFT 论文](https://pmg.csail.mit.edu/papers/osdi99.pdf)  
[FISCO-BCOS的PBFT介绍](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/docs/design/consensus/pbft.html)  
[Liskov对PBFT的讲解](https://www.youtube.com/watch?v=Uj638eFIWg8)  
[What is Tendermint](https://docs.tendermint.com/master/introduction/what-is-tendermint.html)

# 论文笔记
## 一些记录
`3f+1` is the minimum number of replicas that allow an asynchronous system to provide the safety and liveness properties when up to `f` replicas are faulty

在一个由`3f+1`个节点构成的系统中，最多有`f`个恶意节点，才能保证系统的safty和liveness


This many replicas are needed because it must be possible to proceed after communicating with `n-f` replicas, since `f` replicas might be faulty and not responding. However, it is possible that the `f` replicas that did not respond are not faulty and, therefore, `f` of those that responded might be faulty. Even so, there must still be enough responses that those from non-faulty replicas outnumber those from faulty ones, i.e., `n-2f`. Therefore `n > 3f`.

解释了`3f`的原因