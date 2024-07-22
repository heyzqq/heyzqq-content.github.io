---
title: "Hello, hugo!"
date: 2017-09-15T11:30:03+08:00
# weight: 1
# aliases: ["/others"]
tags: ["others"]
author: "Spring"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Install Hugo & simply use it."
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
cover:
    image: "/statics/img/others/girl.png"
    alt: "Hello, hugo~"
    caption: "Hello, hugo~"
    relative: false
    hidden: false
editPost:
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true
---

## 01 Hugo 安装配置

### 1.1 安装 Hugo（Windows）

安装文档：[Install Hugo on Windows](https://gohugo.io/installation/windows/).

使用 scoop 安装，搜索 hugo 安装包如下：

```BAT
> scoop search hugo
Results from local buckets...

Name          Version Source Binaries
----          ------- ------ --------
hugo-extended 0.129.0 main
hugo          0.129.0 main
```

推荐安装扩展版的 hugo：

```BAT
> scoop install hugo-extended
```

### 1.2 创建项目

文档：[Create Site](https://gohugo.io/getting-started/quick-start/).

首先，使用 hugo 创建项目：

```sh
# 可以使用 --format yaml/json 设置配置文件格式，默认 toml
> hugo new site my-site
> cd my-site
```

然后，下载主题并配置：

```sh
# 下载主题，可以使用其他方式，只要把主题放到 themes 目录下即可
> git submodule add https://github.com/nnn/theme-xxx.git themes/xxx
# 配置主题为下载好的那个
> echo "theme = 'xxx'" >> hugo.toml
```

最后，启动项目，访问控制台打印的地址即可：

```SH
> hugo server
Web Server is available at http://localhost:7156/ (bind address 127.0.0.1)
```

## 02 使用模板创建文档（archetypes）

### 2.1 创建文档

要写一篇文章，首先要在 `content` 目录下创建一个文件，例如 `content/hello.md`，然后直接编写该文档即可。

手动创建文件来编写新的文章，其实没有什么问题。但是，如果我们需要写一些通用的、重复的文本，那就得手敲或者复制进去，比较麻烦。

hugo 提供了创建文档的命令 `new content`，该命令会在 `content` 对应的目录下自动创建好文档：

```SH
> hugo new content path/to/article.md
```

使用命令创建文档后，我们直接使用喜欢的 markdown 编辑器编辑该文档即可。

### 2.2 使用模板创建文档

默认的文档，使用了 `archetypes/default.md` 的模板来创建：

```SH
$ cat content/post/test2.md
+++
title = 'article'
date = 2024-07-20T14:00:04+08:00
draft = true
+++
```

如果需要自定义模板，可以在 `archetypes/` 目录下创建自己的模板，例如 `archetypes/tech.md`:

```md
---
title: "My post"
date: 2020-09-15T11:30:03+00:00
tags: ["tech"]
author: "Spring"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
---
```

然后，当需要写 `tech` 类的文章是，使用 `--kind` 参数来指定创建的文档类型即可：

```SH
> hugo new content -k tech posts/java.md
```

## 03 发布

编写好文章后，在当前工程目录下执行 `hugo` 即可自动编译好所有文件，编译好的静态文件都放在 `public` 目录下，只需要把 `public` 内的内容推到 Github 上，或者自己的服务器上即可。

```SH
> hugo
Start building sites …
hugo v0.129.0-e85be29867d71e09ce48d293ad9d1f715bc09bb9+extended windows/amd64 BuildDate=2024-07-17T13:29:16Z VendorInfo=gohugoio


                   | EN
-------------------+-----
  Pages            | 14
  Paginator pages  |  0
  Non-page files   |  0
  Static files     |  3
  Processed images |  0
  Aliases          |  3
  Cleaned          |  0

Total in 63 ms
```

