---
title: "Windows 的端口代理配置（端口映射）"
date: 2023-05-14T20:04:43+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "windows"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "本文介绍了端口映射的用法，通过 netsh 命令可以方便地添加、删除和修改转发规则。此外，还可以使用 netsh 命令查看已存在的端口转发规则和输出配置备份。但需要注意的是，在使用端口映射时需要开启对应端口的入口规则，同时在重置映射（清空）规则时需要慎用。"
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

> 在调试本地程序，或者需要把 WSL 的服务暴露出去，使用端口映射会更加方便。

Windows 的端口映射命令格式如下：

```js
❯ netsh interface portproxy

The following commands are available:

Commands in this context:
show           - Displays information.
add            - Adds a configuration entry to a table.
delete         - Deletes a configuration entry from a table.
dump           - Displays a configuration script.
reset          - Resets portproxy configuration state.
set            - Sets configuration information.
```

`portproxy`：端口代理，它是在强大的网络管理工具 netsh 中，命令本身比较简单，主要就是端口代理的增删改查。


## 01 查看映射（show）

`show` 可以查看已存在的端口转发规则，没有给定参数会输出以下帮助信息：

```js
❯ netsh interface portproxy show

The following commands are available:

Commands in this context:
show all       - Shows all port proxy parameters.
show v4tov4    - Shows parameters for proxying IPv4 connections to another IPv4 port.
show v4tov6    - Shows parameters for proxying IPv4 connections to IPv6.
show v6tov4    - Shows parameters for proxying IPv6 connections to IPv4.
show v6tov6    - Shows parameters for proxying IPv6 connections to another IPv6 port.
```

`show` 命令需要指定查看范围，`all` 表示查看所有，其他的顾名思义表示 ipv4 和 ipv6 之间的转发规则：

```js
❯ netsh interface portproxy show all

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
*               1024        192.168.198.200 1024
```

## 02 添加映射（add）

`add` 用于添加端口转发规则，需要指定 ipv4/ipv6 转发类型：

```js
❯ netsh interface portproxy add help

The following commands are available:

Commands in this context:
add v4tov4     - Adds an entry to listen on for IPv4 and proxy connect to via IPv4.
add v4tov6     - Adds an entry to listen on for IPv4 and proxy connect to via IPv6.
add v6tov4     - Adds an entry to listen on for IPv6 and proxy connect to via IPv4.
add v6tov6     - Adds an entry to listen on for IPv6 and proxy connect to via IPv6.
```

具体用法如下：

```js
❯ netsh interface portproxy add v4tov4 help

Usage: add v4tov4 [listenport=]<integer>|<servicename>
             [connectaddress=]<IPv4 address>|<hostname>
            [[connectport=]<integer>|<servicename>]
            [[listenaddress=]<IPv4 address>|<hostname>]
            [[protocol=]tcp]

Parameters:

       Tag              Value
       listenport     - IPv4 port on which to listen.（必须）
       connectaddress - IPv4 address to which to connect.（必须）
       connectport    - IPv4 port to which to connect.（必须）
       listenaddress  - IPv4 address on which to listen.（可选，默认所有 *）
       protocol       - Protocol to use.  Currently only TCP is supported.（目前仅支持 TCP，不用填）
```

**示例**：把端口 1025 的流量转发到 wsl 的 1025 端口上，让外部机器能直接访问到 wsl 子系统（默认到 `0.0.0.0`）

```js
❯ netsh interface portproxy add v4tov4 listenport=1025 connectaddress=192.168.198.200 connectport=1025

❯ netsh interface portproxy show all

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
*               1024        192.168.198.200 1024
*               1025        192.168.198.200 1025
```

- 前提：防火墙需要开启 1024 入口规则
- 1）本地监听 1025 端口
- 2）把 1025 端口连接到 192.168.198.200 的 1025 端口，这样，请求本机 1025 的流量，都会转到 wsl 的 1025

## 03 删除映射（delete）

`delete` 同样要指定 ip 类型，删除时只需指定监听的地址和端口即可：

- listenaddress：监听地址，可选（默认 \*）
- listenport: 监听端口，必选

```js
❯ netsh interface portproxy delete v4tov4 listenport=1025

❯ netsh interface portproxy show all

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
*               1024        192.168.198.200 1024
```

## 04 输出配置（备份，dump）

`dump` 命令输出已配置的转发规则，用于备份已存在的规则，可以重定向到文件保存起来。

```SH
❯ netsh interface portproxy show all

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
0.0.0.0         1024        192.168.198.200 1024

❯ netsh interface portproxy dump

#========================
# Port Proxy configuration
#========================
pushd interface portproxy

reset
add v4tov4 listenport=1024 connectaddress=192.168.198.200 connectport=1024


popd

# End of Port Proxy configuration
```

示例：备份到文件（没有找到还原方式，官方文档没有 dump 参数，有点尬。。。）

```sh
# 备份
> netsh interface portproxy dump > portproxy.bak
```

## 05 修改或添加（set）

`set` 命令可以修改或新增转发规则。

新增规则：

```js
❯ netsh interface portproxy set v4tov4 listenaddress=0.0.0.0 listenport=1024 connectaddress=192.168.198.200 connectport=2048

❯ netsh interface portproxy show all

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
*               1024        192.168.198.200 1024
0.0.0.0         1024        192.168.198.200 2048
```

修改规则：

```js
❯ netsh interface portproxy set v4tov4 listenaddress=0.0.0.0 listenport=1024 connectaddress=192.168.198.200 connectport=2049

❯ netsh interface portproxy show all

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
*               1024        192.168.198.200 1024
0.0.0.0         1024        192.168.198.200 2049
```

## 06 重置映射（清空，reset）

清空所有规则，慎用：

```sh
# 重置
❯ netsh interface portproxy reset
# 查看
❯ netsh interface portproxy show all
```