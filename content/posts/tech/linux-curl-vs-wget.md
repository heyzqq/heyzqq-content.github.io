---
title: "curl 于 wget 的区别（下载文件）"
date: 2021-05-11T20:49:41+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "linux"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "wget 是个专职的下载利器，简单，专一，极致；curl 可以下载，但是长项不在于下载，而在于模拟提交 web 数据，调试网页等。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
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

## 01 curl 和 wget 的区别

> wget 是个专职的下载利器，简单，专一，极致；  
curl 可以下载，但是长项不在于下载，而在于模拟提交 web 数据，POST/GET 请求，调试网页，等等。

## 02 具体使用（下载）

1. 下载文件

```sh
curl -O http://man.linuxde.net/text.iso                    # O大写，不用O只是打印内容不会下载
wget http://www.linuxde.net/text.iso                       # 不用参数，直接下载文件
```

2. 下载文件并重命名

```sh
curl -o rename.iso http://man.linuxde.net/text.iso         # o小写
wget -O rename.zip http://www.linuxde.net/text.iso         # O大写
```

3. 断点续传

```sh
curl -O -C - http://man.linuxde.net/text.iso               # O大写；C大写，- 表示不指定续传的偏移量，默认从本地文件计算
wget -c http://www.linuxde.net/text.iso                    # c小写
``` 

4. 限速下载

```sh
curl --limit-rate 50k -O http://man.linuxde.net/text.iso
wget --limit-rate=50k http://www.linuxde.net/text.iso
``` 

5. 显示响应头部信息

```sh
curl -I http://man.linuxde.net/text.iso
wget --server-response http://www.linuxde.net/test.iso
```

6. wget 利器 → 打包下载网站

```sh
wget --mirror -p --convert-links -P /var/www/html http://man.linuxde.net/
```

------------------------------------

## References

[1] https://www.zhihu.com/question/19598302  
[2] https://www.cnblogs.com/lsdb/p/7171779.html  
