---
title: "使用 Docker-Compose 搭建 Maven 私服 Nexus"
date: 2023-06-30T20:40:04+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "docker", "maven"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Maven 私服 Nexus 的搭建。"
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

## 01 搭建私服的初衷

### 1.1 为什么搭建私服？

使用 Maven 管理项目依赖时，通常会从 Maven 仓库拉取第三方依赖，此时只需要配置公共仓库源即可。

但是，如果需要使用公司内部依赖，需要同时使用第三方依赖和私有依赖，这就需要搭建私服，用以存储私有组件库。

搭建私服，可以实现如下功能：

1. 共享私有依赖，保护敏感信息
2. 保存依赖版本，避免依赖变化或升级受影响
3. 提升代理速读，缓存常用依赖到私服
4. 自定义配置策略、规则、权限等

### 2.2 为什么选择 Nexus ？

常见仓库管理有如下几种：

1. **Nexus Repository Manager**：Nexus 是一个广泛使用的 Maven 仓库管理器，它提供了丰富的功能，包括支持 Maven、Gradle、Ivy 等构建工具，具有强大的缓存和代理能力，以及用户权限控制、部署规则等功能。

2. **Artifactory**：Artifactory 是另一个流行的 Maven 仓库管理器，它支持多种构建工具，提供了企业级的功能，例如高度可定制的权限控制、智能缓存和复制、跨数据中心复制、虚拟仓库等。

3. **Archiva**：Archiva 是一个轻量级的 Maven 仓库管理器，它提供了基本的 Maven 仓库功能，例如部署、下载和缓存。虽然功能相对较少，但易于安装和使用，适用于小型项目或简单的私有仓库需求。

选择 Nexus 一来是满足需求，需要的功能都包含，二来是比较熟悉。其他管理系统也都可以，视情况选择即可。

## 02 使用 Docker-Compose 搭建

### 2.1 Docker-Compose 搭建

首先，编写 `docker-compose.yaml`：

```yaml
version: '3.9'
services:
  nexus:
    image: sonatype/nexus3:latest
    #image: sonatype/nexus3:3.56.0
    container_name: nexus3
    restart: always
    ports:
      - 10240:8081
    volumes:
      - ./nexus-data:/nexus-data
```

然后，创建目录 `./nexus-data` 映射数据（不需要备份的就不需要啦），因为需要配置目录权限，所以这里要手动创建目录：

```SH
$ mkdir nexus-data
$ chown -R 200 ./nexus-data
```

这里修改目录所有者，是因为容器创建了 nexus 用户，uid 为 200，如果不修改，启动容器时，会报各种无权限错误：

```SH
# 在容器内查看
bash-4.4$ cat /etc/passwd
# ……
nexus:x:200:200:Nexus Repository Manager user:/opt/sonatype/nexus:/bin/false
```

最后启动，等待启动成功即可：

```SH
$ docker-compose up -d
```

### 2.2 Nexus 配置

#### 2.2.1 登录系统

上述配置映射了 `10240` 端口，所以访问 `http://ip:10204`，使用 admin 登录，nexus3 密码在 `nexus-data/admin.password`

```
$ cat ./nexus-data/admin.password
9233-22...
```

#### 2.2.2 配置仓库

首先，顶部选择 【⚙】-> Repository -> Repositories -> ＋ Create Repository

然后，Recipe 选择 maven2，根据情况选择

- maven2 (group)：仓库组，也就是「第三方＋私有」仓库
- maven2 (hosted)：私服本地，「私有仓库」，私人上传的组件、依赖
- maven2 (proxy)：代理，「第三方仓库」，阿里云、华为云等

##### 私服配置

私服（hosted）配置，只需要填写名称即可。

但是，由于私服存在很多不同阶段的开发依赖版本，所以最好是创建多个仓库，分别存放开发版、稳定版的依赖。

【Version policy】可以选择 `Release` 和 `Snapshot`，当然如果只是简单的测试，直接选择 `Mixed` 混合模式即可，这样就可以上传任何阶段的依赖了。

##### 代理配置

代理（proxy）配置，只需要填写名称和地址即可。

【Remote storage】填写 URL，如 `https://maven.aliyun.com/repository/public`。

然后点击【Create repository】即可。

##### 配置仓库组

仓库组（group）配置，只需要填写名称，然后选择仓库进行组合即可。

由于开发需要依赖内部 snapshot 版本依赖，所以一般【Version policy】选择 `Mixed`。

在 Group ->【Member repositories】中选择需要的仓库，组合起来，保存即可。

然后，在 Maven 的 settings.xml 只需要配置该仓库组，就可以同时使用多个仓库了。

#### 2.2.3 配置角色和用户

角色和用户配置相对比较简单，自行测试即可。

主要是创建角色，分配仓库增删改查权限。然后创建用户，分配角色，就是这么简单的。

## References

[1] 云中月. docker 中使用 nexus 镜像搭建 maven 私服. 2023-06-06.  
[2] Aqoo. 如何搭建 maven 私有仓库. https://juejin.cn/post/7231902909993631804, 2023-05-11.  
