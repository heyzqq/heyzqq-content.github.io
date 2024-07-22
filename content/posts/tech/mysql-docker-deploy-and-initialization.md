---
title: "MySQL 的容器化部署与数据库初始化"
date: 2023-05-20T12:15:27+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "MySQL"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "本文介绍了如何在 Docker 中安装 MySQL，并通过制作镜像、使用 Docker Compose 管理容器等方式来管理 MySQL 容器。在 Docker 中启动 MySQL 容器时，会对目录 /docker-entrypoint-initdb.d 下的 .sh、.sql 和 .sql.gz 类型的文件进行初始化。通过编写 Dockerfile 文件，可以将要执行的 SQL 语句放到镜像中进行管理；通过 Docker Compose 可以管理容器，其中配置文件 docker-compose.yaml 可以配置重启策略、环境变量和映射等参数。最后还演示了如何通过命令行和 Docker Compose 启动容器。"
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

## 01 概述

### 1.1 环境

CentOS 8

```sh
$ docker --version
Docker version 20.10.11, build dea9396
$ docker-compose --version
Docker Compose version v2.16.0
```

### 1.2 Docker 安装 MySQL 的初始化原理

MySQL Docker 官方介绍：[Initializing a refsh instance](https://hub.docker.com/_/mysql)

> When a container is started for the first time, a new database with the specified name will be created and initialized with the provided configuration variables. Furthermore, it will execute files with extensions ++.sh, .sql and .sql.gz++ that are found in **/docker-entrypoint-initdb.d**. Files will be executed **in alphabetical order**. You can easily populate your mysql services by mounting a SQL dump into that directory and provide custom images with contributed data. SQL files will be imported by default to the database specified by the **MYSQL_DATABASE** variable.

即，容器首次启动后，会做如下初始化动作：

1. 加载目录为 `/docker-entrypoint-initdb.d`，后缀为 `.sh`, `sql` 和 `.sql.gz` 的文件
2. 按照文件名的字母顺序，依次执行
3. 默认情况下，SQL 文件将导入到 `MYSQL_DATABASE` 指定的数据库中（*sh 格式暂不了解作用*）

## 02 制作镜像

### 2.1 编写 Dockerfile

不一定要创建新的镜像，只是在项目部署时，习惯使用自己构建的镜像，定制一点配置，这里只是把初始化 sql 放到镜像中。

创建一个新的目录，结构如下：

```bash
- db/            # 存放 SQL 文件
- Dockerfile    
```

Dockerfile 文件如下：

```bash
FROM mysql:8.0.33
MAINTAINER springx.fun

COPY ./db/*.sql /docker-entrypoint-initdb.d/
```

### 2.2 制作镜像

执行 build 制作镜像，自动拉取依赖镜像，并制作自定义镜像：

```bash
$ docker build -t springx.fun/mysql:8.0.33 .
Sending build context to Docker daemon   5.12kB
Step 1/3 : FROM mysql:8.0.33
8.0.33: Pulling from library/mysql
328ba678bf27: Pulling fs layer 
f3f5ff008d73: Pulling fs layer 
dd7054d6d0c7: Downloading 
70b5d4e8750e: Waiting 

...
```

参数说明：

- `-t`：指定构建的镜像名称和标签。例如 `-t myapp:latest`，构建出来的镜像名称为 myapp，标签为 latest，这里指定 8.0.33
- `.`：Dockerfile 的所在路径或文件名。这里指定当前路径，他会自动找到 `Dockerfile` 并构建

**为什么要手动制作镜像？**

其实如果要使用 Docker Compose，是不用自己手动构建镜像的。但是有遇到过一次异常，Docker Compose 构建的镜像与预期不符，没有可执行的命令，所以紧急情况下手动构建处理（当时是 Docker 19.03.4，可能与 Docker Compose v2.17.2 版本有关？）

## 03 使用 Docker Compose 管理容器

### 3.1 Docker Compose Yaml 配置

编写 `docker-compose.yaml` 文件：

```yaml
version: '3.9'

services:
  mysql:
    # 运行起来的容器名称
    container_name: mysql-test
    # 自定构建镜像的镜像名称和标签
    image: springx.fun/mysql:8
    # 构建镜像的配置
    build:
      # Dockerfile 所在路径
      context: ./
      # 还可以指定其他参数
    # 重启策略：除非用户主动停止，否则出错就一直重启
    restart: unless-stopped
    # 端口映射：宿主端口:容器端口
    ports:
      - 23306:3306
    # 环境变量
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE:-springx}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-aaa123}
      TZ: Asia/Shanghai
    # 映射：把必要数据映射到指定位置
    volumes:
      - ./mysql/logs/:/logs/
      - ./mysql/data/:/var/lib/mysql/
      - ./mysql/db/:/docker-entrypoint-initdb.d/
      - ./mysql/conf.d/:/etc/mysql/conf.d/
    # Docker 日志限制，否则会无限大，占据过多磁盘
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 3.2 容器重启策略

Dcoker 容器的重启策略（restart）：

- `no`：默认，容器退出不重启
- `on-failure`：失败重启，次数可自定义
- `always`：容器退出时，总是重启
- `unless-stopped`：容器退出时重启，除非手动停止

Docker Compose Version 3 配置如下（See [restart policy](https://docs.docker.com/compose/compose-file/compose-file-v3/#restart_policy)）：

```YAML
version: "3.9"
services:
  mysql:
    image: mysql:alpine
    deploy:
      restart_policy:
        condition: on-failure  # 失败重启
        delay: 5s              # 尝试间隔    
        max_attempts: 3        # 最大尝试次数
        window: 120s           # 在决定重启是否成功之前需要等待多长时间
```

### 3.3 MySQL 的环境变量

Docker Compose 支持 `${ENV:-DEFAULT}` 格式，从环境变量中获取值，一些通用配置、密码等一般都可以通过这种方式配置。

- `ENV`：环境变量，既可以指定环境文件，也可以记录在默认的 `.env` 文件中（当前目录下）
  ```txt
  ENV = XXYY
  ```
- `DEFAULT`：默认值，如果读取不到 `ENV`，值使用默认值填充。`-` 为固定写法。

#### MYSQL_ROOT_PASSWORD

`MYSQL_ROOT_PASSWORD` 配置 root 用户的初始密码。

正式环境最好还是修改一下。

#### MYSQL_DATABASE

`MYSQL_DATABASE` 配置默认数据库，MySQL 容器启动后，会自动创建该数据库，并且初始化的 sql 默认导入该库中。


## 04 启动容器

### 4.1 Docker 直接启动

使用 Docker 直接运行，需要注意，部分参数需要注意顺序，比如环境变量，需要在指定镜像之前：

```
$ docker run \
    -e "MYSQL_ROOT_PASSWORD=ttt123" \
    -e "MYSQL_DATABASE=test" \
    springx.fun/mysql:8 \
    --name mysql-test \
    -d                                  
```

### 4.2 Docker Compose 启动

```
# 启动
$ docker-compose up -d
# 查看容器运行情况
$ docker-compose ps -a
# 查看日志，看看启动是否有误
$ docker-compose logs
```

当 `up` 命令执行，Docker Compose 会自动执行 build 命令，构建相应的镜像，然后使用该镜像启动一个容器。


