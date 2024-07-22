---
title: "如何为终端或 Git 设置代理？"
date: 2023-07-22T23:15:37+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "proxy"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "在 Windows 中使用 CMD、PowerShell 和 Git Bash / Linux 终端设置代理的方法，以及如何在 SSH 方式下配置代理。"
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

## 碎碎念

由于网络原因，我们不得不使用一些手段，以疏通或者加速它。

在 CMD/PowerShell 终端配置代理，可以允许整个命令行得到代理。

使用 git config 配置代理，则仅支持 git 自身获得代理。

## CMD 设置代理

Windows 的 cmd 使用 `set` 命令来配置：

```bash
> set http_proxy=http://127.0.0.1:1080
# Or
> set https_proxy=https://127.0.0.1:1080
```

这种方式，可以进行终端全局的代理。

Windows 还有 PowerShell 终端，也是类似的。

## PowerShell 设置代理

PowerShell 配置方式比较特殊，但同样是设置全局变量 `HTTP_PROXY/HTTPS_PROXY`（不区分大小写）：

```bash
> $env:HTTP_PROXY="http://127.0.0.1:1080"
# Or
> $env:HTTPS_PROXY="https://127.0.0.1:1080" 
```

**注：** 终端配置的方式，每次关闭之后，都要重新配置。除了写一段初始化脚本，还可以直接配置 git 的全局代理。

## Git Bash 设置代理

在 Git Bash 中，可以通过 `git config` 设置 HTTP 代理：

```SH
> git config --global http.proxy "http://127.0.0.1:1080"
# Or
> git config --global https.proxy "https://127.0.0.1:1080"
```

但是，这种方式仅适用于 HTTP/HTTPS，不适用于 SSH 方式（即 `git clone git@github.com:username/repo.git`）。

如果要使用 SSH 方式通信，需要配置 Git 的 `~/.ssh/config`。

**注意**

直接配置 git 的代理，会永久保存，所有 git 连接都走代理。如果代理服务器不在线，会导致 git 无法推送/拉取，需要重置代理配置：

```SH
> git config --global --unset http.proxy
```

## SSH 方式代理配置

使用 https 的方式有个缺点，就是输入账号密码，而使用 ssh 方式可以自动使用密钥验证。

打开 `~/.ssh/config` 添加配置（不存在则创建一个）：

```
Host github.com
   User Spring                                        # 用户名
   IdentityFile "C:\Users\springx\.ssh\id_rsa"        # 私钥
   ProxyCommand "D:\OpenSSH-Win64\bin\nc.exe" -X 5 -x 127.0.0.1:1080 %h %p   # 连接代理
```

`ProxyCommand` 是 OpenSSH 配置文件中用于指定在建立 SSH 连接时，通过代理服务器进行连接的命令。它允许你在 SSH 客户端和 SSH 服务器之间建立一个额外的中间连接，通过该中间连接来传输 SSH 流量。

这里使用 nc（ncat - Concatenate and redirect sockets） 网络工具，CentOS 直接安装 nc，Ubuntu 是 netcat，Windows 下可以安装 OpenSSH，或者借用其他程序携带的 nc.exe（我使用的是 MobaXterm 内置的 nc.exe）

- `X`：socks 版本号（代理服务器的类型，常见的类型包括 http、https 和 socks5）
  - HTTPS：没有连接，暂时无法验证
  - HTTP：`ProxyCommand nc -X connect -x <HOST>:<PORT> %h %p` （不用 -X 参数可以连接，未验证）
- `x`：指定代理服务器 ip 和端口号
- `%h %p`：变量，分别表示 github 的 host 和 ssh 端口号（默认 22）

## Linux

在 Linux 中为命令行设置代理，与 Windows 的 CMD 方式一致：

```SH
> export http_proxy=http://127.0.0.1:1080
# Or
> export https_proxy=https://127.0.0.1:1080
```

## QA?

* 仅设置 http 代理就可以正常访问 https，这是代理本身提供 http 的原因？

这里的 HTTP 代理并不是指要代理 HTTP 协议的请求，而是指数据从软件（浏览器）到代理客户端的通信方式，本机的代理客户端开启的连接方式为 HTTP，这中间的数据是明文传输的，有被泄露的风险。

而 HTTPS 代理的核心思路，就是使用 https/tls 对浏览器到代理这一段的通信进行加密，这样中间节点就不能监听数据。

这里参考：[“允许来自局域网的连接” 到底怎么用](https://moe.best/gotagota/ss-ssr-allow-lan.html)

## References

[1] FrozenMap. 为 git bash 设置代理. https://jjayyyyyyy.github.io/2019/08/11/git_bash_proxy.html, 2019-08-11.  
[2] 咕咕. ssh 的高级用法 - ProxyCommand. https://bugwz.com/2019/10/09/ssh-proxycommand/, 2019-10-09.  

*- 2023-12-05 edited -*
