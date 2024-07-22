---
title: "GBK 和 Unicode 之间的转换问题——“锟斤拷”的由来"
date: 2022-05-25T21:38:17+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "java"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "本文主要介绍了 GBK 和 Unicode 之间的转换问题。其中，当使用错误的编码方式进行转换时，会产生乱码现象，例如将正常的 GBK 字节流以 UTF-8 解码或将正常的 UTF-8 字节流以 GBK 解码。此外，在某些场景中，由于中途被改变，导致输出的字符串出现乱码，例如在 HTTP 请求回来默认使用 UTF8 生成字符串。"
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

## GBK 和 Unicode 之间的转换问题

### 1. GBK 被解密为 UTF-8

正常的 `GBK` 字节流，以为是 `UTF-8`，所以用 `UTF-8` 去解码：

- 输出特点：一堆的黑色菱形+问号
- 输出示例：�Ҳ�����

```java
@Test
public void test() throws UnsupportedEncodingException {
    String str = "我不是锟斤拷";

    byte[] buff = str.getBytes("GBK"); // 这里只要不抛异常，数据一定不会被破坏
    String str1 = new String(buff, "UTF-8");
}
```

### 2. UTF-8 被解密为 GBK

正常的 `UTF-8` 字节流，以为是 `GBK`，所以用 `GBK` 去解码：

- 输出示例：鎴戜笉鏄敓鏂ゆ嫹

```JAVA
@Test
public void test() throws UnsupportedEncodingException {
    String str = "我不是锟斤拷";

    byte[] buff = str.getBytes("UTF-8"); // 这里只要不抛异常，数据一定不会被破坏
    String str1 = new String(buff, "GBK"); // 这里破坏了
}
```

### 3. "锟斤拷"的由来

正常的 `GBK` 字节流，中途被 `UTF-8` 解码了，又用 `GBK` 解码一遍：

- 输出特点：锟斤拷锟斤拷
- 输出示例：锟揭诧拷锟斤拷锟斤拷

```JAVA
@Test
public void test() throws UnsupportedEncodingException {
    String str = "我不是锟斤拷";

    byte[] gbk = str.getBytes("GBK"); // 原来的 GBK
    str = new String(gbk, "UTF-8");   // 中途被转为 UTF-8
    String str1 = new String(str.getBytes(StandardCharsets.UTF_8), "GBK");
}
```

### 总结分析

1. 情况 1、2 是使用错误编码方式
2. 情况 3 是中途被改变的（如 HTTP 请求回来，默认用 `UTF8` 生成字符串，此时就被转码了）

## Reference

[1] pollyduan. 一段 java 代码带你认识锟斤拷[EB/OL]. https://cloud.tencent.com/developer/article/1532357, 2019-11-04.  
