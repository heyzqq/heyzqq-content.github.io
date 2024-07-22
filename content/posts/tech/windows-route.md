---
title: "Windows 的路由配置"
date: 2023-05-14T20:59:53+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "windows"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "这篇文章介绍了 Windows 操作系统中的路由命令，包括路由表的增删改查操作。使用 route print 命令可以查看所有路由或指定相应条件的路由，使用 route add 命令可以添加默认路由、指定 IP 的路由以及指定网段的路由，使用 route delete 命令可以删除指定 IP 或网段的路由，使用 route change 命令可以修改网关和跃点数。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
editPost: # github 当前文章修改/建议
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true # to append file path to Edit link
---

## 00 Windows 下的路由

路由是什么？
> 路由是指在计算机网络中将数据包从源地址传输到目标地址的过程，通过路由器根据网络拓扑和路由表中的信息进行决策和转发。


Windows 的路由命令格式如下：

> Manipulates network routing tables. 操作网络路由表

```sh
❯ route /?

ROUTE [-f] [-p] [-4|-6] command [destination]
                  [MASK netmask]  [gateway] [METRIC metric]  [IF interface]
```

- `-f`：清除所有路由表项。如果与其中一个命令一起使用，则在运行命令之前清除表。
- `-p`：与 `ADD` 命令结合使用，标识添加为永久路由
- `METRIC`：跃点数，网络接口顺序，值越小越优先（会自动调整）
- `command`：命令，有四个可选命令，分别对应路由的增删改查
  - `PRINT`：打印路由
  - `ADD`：添加路由
  - `DELETE`：删除路由
  - `CHANGE`：修改路由

## 01 查看路由表（print）

查看路由的命令格式如下：

- `route print`：查看所有
- `route print -4`：查看 ipv4 路由表
- `route print -6`：查看 ipv6 路由表
- `route print 192.168.*`：查看匹配的路由

示例：查看 `172.16.30` 相关的路由信息

```sh
❯ route print -4 172.16.30*
===========================================================================
Interface List
 48...00 15 00 00 00 00 ......Hyper-V Virtual Ethernet Adapter
 12...50 eb 00 00 00 00 ......Intel(R) Wireless-AC 9462
  1...........................Software Loopback Interface 1
===========================================================================

IPv4 Route Table
===========================================================================
Active Routes:      # 动态路由表
Network Destination        Netmask          Gateway       Interface  Metric
      172.16.30.0    255.255.254.0         On-link      172.16.30.76    291
     172.16.30.76  255.255.255.255         On-link      172.16.30.76    291
===========================================================================
Persistent Routes:  # 永久路由表
  None
```

- On-link：在链路上，即目标 ip（Network Destination）与接口（Interface）在同一网段
- Interface List 中，前面的数字为「接口号码」，接下来是 MAC 地址，后面是接口名称

## 02 添加路由（add）

添加路由的命令格式如下，如果没有给定 IF，则默认为 gateway 查找一个最合适的接口：

```js
> route ADD 157.0.0.0 MASK 255.0.0.0  157.55.80.1 METRIC 3 IF 2
         destination^      ^mask      ^gateway     metric^    ^
                                                     Interface^
```

- `IF interface` 的 interface 代表接口号码，可以在 `route print` 的 Interface List 中查看号码。

### 2.1 添加默认路由

```js
> route add 0.0.0.0 mask 0.0.0.0 172.16.30.1
OK!
```

### 2.2 指定 ip 路由

指定 ip 不可指定 mask，默认 255.255.255.255

```sh
> route add 192.168.199.200 172.16.30.1 IF 10
OK!
> route print -4 192.168.199.200
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
    192.168.199.200    255.255.255.0      On-link      172.16.30.1    108
```

### 2.3 指定网段路由

添加网段时不能填写具体 ip，网段取值为 0（如 192.168，写成 192.168.0.0）

```js
> route add 192.168.199.0 mask 255.255.255.0 172.16.30.76 IF 10
OK!
> route print -4 192.168.199.*
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
    192.168.199.0      255.255.255.0      On-link      172.16.30.76    108
```

## 03 删除路由（delete）

删除路由只需指定目标 ip 或网段即可：

```sh
❯ route print -4 192.168.188.*
IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
    192.168.188.0    255.255.255.0         On-link      172.16.30.76    108
  192.168.188.255  255.255.255.255         On-link      172.16.30.76    291
  
❯ route delete 192.168.188.255
 OK!

❯ route print -4 192.168.188.*
IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
    192.168.188.0    255.255.255.0         On-link      172.16.30.76    108
```

## 04 修改路由（change）

路由修改仅支持更改网关（gateway）和跃点数（metric）

```js
> route change 192.168.199.200 192.168.199.1 mertic 108
OK!
```

## REFERENCES

[1] ileeoyo. Windows 路由表“在链路上”. <https://ileeoyo.gitee.io/post/windows%E8%B7%AF%E7%94%B1%E8%A1%A8%E5%9C%A8%E9%93%BE%E8%B7%AF%E4%B8%8A/>, 2020-05-25.
