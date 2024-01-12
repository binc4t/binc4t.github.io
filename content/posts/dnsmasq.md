+++ 
draft = false
date = 2024-01-12T23:04:41+08:00
title = "dnsmasq 使用备忘"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

最近在给手机测试时，想要把手机上对特定域名的请求劫持到Mac上来，就需要用到dnsmasq，把Mac变成一个dns服务器，然后手机的dns服务器地址修改到mac上来。下面做一个配置备忘

参考：https://blog.niekun.net/archives/1869.html

省略安装步骤，debian用 `apt install`，mac用 `brew install` 即可，下面进行配置

dnsmasq会按照一定的顺序去进行域名解析

```bash
首先是系统以及自定义的hosts文件： /etc/hosts  /etc/hosts.dnsmasq

然后是上游dns server，定义在：resolv.dnsmasq.conf 
```

默认dns server记录在`/etc/resolv.conf`文件中，既然我们要使用dnsmasq来做dns服务器，那么就应该让dnsmasq能够查询系统原本的dns sever，等于是用dnsmasq做了一个中继

```bash
sudo cp /etc/resolv.conf /etc/resolv.dnsmasq.conf
```

然后在/etc/resolv.conf文件中，将内容改为

```bash
nameserver 127.0.0.1
```

这样就把本机的dns server指向了 127.0.0.1，这时候机器的dns请求就会发往127.0.0.1的53端口，dns默认监听端口，也是dnsmasq的监听端口

然后再在 `/etc/dnsmasq.conf` 文件中输入配置

```bash
# 监听地址：
# 如果只写 127.0.0.1 则只处理本机的 DNS 解析，不写这句默认监听所有网口
# listen-address=127.0.0.1,192.168.8.132

# 指定自定义 hosts 文件：
addn-hosts=/etc/hosts.dnsmasq

# 指定上游 DNS 服务列表的配置文件
resolv-file=/etc/resolv.dnsmasq.conf

# 按照 DNS 列表一个个查询，否则将请求发送到所有 DNS 服务器
strict-order

# 表示对下面设置的所有 server 发起查询请求，选择响应最快的服务器的结果
all-servers
```

这样配置过程就完成了！

此时本机的dns请求的查询路径就会是

```bash
/etc/hosts -> dnsmasq
```

然后dnsmasq又会依次检索

```bash

/etc/hosts  /etc/hosts.dnsmasq -> resolv.dnsmasq.conf
```

那么此时我们在 `/etc/hosts` 或者 `/etc/hosts.dnsmasq` 文件中定义一条 host，就可以改变dns解析的结果

不仅是本机的dns查询，因为dnsmasq监听53端口，相当于本机也是一个dns服务器，其他服务器可以向本机进行dns查询，或者把dns服务器地址设置成本机，这在进行手机的dns劫持时很有用处