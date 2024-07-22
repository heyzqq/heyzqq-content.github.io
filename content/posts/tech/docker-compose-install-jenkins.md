---
title: "使用 Docker-Compose 搭建 Jenkins"
date: 2023-08-21T21:39:09+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "docker", "jenkins"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "这篇文章介绍了使用 Docker 搭建 Jenkins 的主要安装步骤和权限问题处理。对于 Docker 权限问题，可以将容器内部的用户添加到宿主机的 Docker 组中；对于映射目录权限问题，可以手动创建目录并分配权限，或者修改已创建目录的权限。"
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
## 00 碎碎念

**为什么使用 Docker 去搭建 Jenkins 呢？**

主要原因是个人频繁切换环境，有时候在本地虚拟机玩，有时候会在云服务器上折腾，渐感被自己折磨过多了，因此希望能够创建一个配置文件，以便一键启动并快速方便地使用。另外，自己经常在其他地方装环境，公司的服务也比较老，一但要迁移，用 Docker 就很方便。所以，使用 Docker 是很适合的。

**网上那么多一样的文章，为什么要写这篇呢？**

虽然说这篇文章是作为笔记的，但是因为自己在使用网上的方法安装的时候，遇到几个问题，尝试了很多方法才成功，所以想要记录一下，以备不时之需。

## 01 Docker Compose 安装

以下是测试过的，能正常部署和使用 Jenkins 的 Docker Compose 配置，部分参数请按需调整。

```YAML
version: '3.9'
services:
  jenkins:
    # 镜像选择了 jdk17 最新版，可自行选择
    image: jenkins/jenkins:jdk17
    container_name: jenkins
    # 重启策略：除非手动停止，否则出错会无限重启
    restart: unless-stopped
    # 指定用户 uid:gid（`用户id`:`宿主机的docker组id`，跟文件权限有关）
    user: 1000:995
    ports:
      # 8080 为 Jenkins 的 Web 端口
      - 9001:8080
      # 50000 为代理节点与主服务器的通信端口？
      - 50001:50000
    volumes:
      # 同步宿主机的时间
      - /etc/timezone:/etc/timezone
      - /etc/localtime:/etc/localtime
      # Jenkins 数据目录映射出来，方面操作和备份
      - .jenkins_home:/var/jenkins_home
      # 把宿主机的 docker 和 docker-compose 给 Jenkins 使用，这样可以直接在 Jenkins 内部打镜像，并直接操作容器
      - /usr/bin/docker:/usr/bin/docker
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/local/bin/docker-compose:/usr/local/bin/docker-compose
```

使用 Docker Compose 直接启动即可：

```SH
# -d：后台启动
$ docker-compose up -d

# 1. 查看日志，临时登录密码会放到文件中，也会直接打在日志上
#    临时密码位置：/var/jenkins_home/secrets/initialAdminPassword
# 2. 或使用 docker 命令查看： docker logs jenkins
$ docker-compose logs -f

Jenkins initial setup is required.An admin user has been created and a password generated.
Please use the following password to proceed to installation:

# 临时密码
57ea14a04f9464d895655348d6ad6a86
```

然后，访问 9001（这里已经映射为 9001 端口了）：`http://ip:9001`，输入临时密码登录。

接着，选择安装插件，创建用户，等待完成即可。（这些步骤都可以跳过，后续再进行即可）

## 02 问题及解决办法

### 2.1 Docker 权限问题（Permission denied）

**用网上的方法，折腾了好久……**

把 Docker 相关文件映射到容器内部后，会遇到权限问题（**如果直接指定 root 用户部署，不会有权限问题！**），网上有许多把 jenkins 用户添加到 docker 组的方式，但这依然是基于宿主机的环境，未必能直接解决。

我比较爱折腾，环境稍微有差异，直接把 jenkins 用户添加到 docker 组依然有权限问题，接下来看看具体的情况和处理方法。

**事情是这样子的~**

首先，来看看 `docker.sock` 在 Jenkins 容器内部的情况：

```SH
jenkins@b0e6d3e8ce07:~$ ls -l /var/run/ | grep docker
srw-rw----. 1 root  998 0 Jun  3 01:59 docker.sock
```

`docker.sock` 是我们从宿主机映射到容器内的 sock 文件，它是 Docker 守护进程（Docker daemon）与 Docker 客户端之间进行通信的 UNIX 套接字文件（UNIX socket file）。

由于 docker 相关文件是属于宿主机的，原则上不直接修改它，而是通过修改容器来兼容。但是，这里的 sock 属于 root 用户，如果使用普通用户（默认 jenkins:1000）部署，并没有权限去读取。

`998` 是 docker 组的组 id（gid），网上找到比较好的方法是，把 jenkins 用户直接加入该组，就可以正常使用宿主机 docker 了。

而我的问题就出在这里。

**先用网上的方法试试？**

接着，来看下容器内部的情况：

```SH
jenkins@b0e6d3e8ce07:~$ id jenkins
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)

jenkins@b0e6d3e8ce07:~$ id
uid=1000(jenkins) gid=995 groups=995

# 下面是配置 `user: root` 部署的，容器内部使用 root 用户，可以正常使用 docker
root@8900354a95f1:/# id
uid=0(root) gid=0(root) groups=0(root)
```

第一个命令，`id jenkins` 表明 jenkins 用户的用户 id 和组 id 都为 1000；  
第二个命令，`id` 表明当前登录会话的用户 id 为 1000，而组 id 和 groups 为 995（为什么不一样？因为 docker-compose 配置文件设置的，**groups 就是重点**）。

也就是说，当前登录会话的用户，既不是 root 用户，也不属于 998 组，所有没有权限读取 `docker.sock`。

所以，到这儿得先把 jenkins 加入 998 组，否则不可能有权限的。这个 docker 组应该不存在，我们需要先创建该组，然后再把 jenkins 加入：

```SH
# 先退出容器，使用 root 登录（-u 指定登录用户）
$ sudo docker exec -it -u root jenkins bash

# 然后，创建 docker 组，指定与 docker.sock 一样的组 id
# groupadd -g <gid> <group_name>
$ groupadd -g 998 docker

# 最后，把 jenkins 加入该组
# usermod -a -G <group_name> <username>
$ usermod -aG docker jenkins
```

到这一步，我还是没有权限，重启容器也没有用，这是为什么呢？

**一路走到黑？**

折腾了很久都没效果，本来已经要放弃了，就在宿主机试验了一遍：docker 组已经存在，把普通用户加入 docker，重新登录，普通用户居然有权限了！

```SH
[springx@** ~]$ id
uid=1000(springx) gid=1000(springx) groups=1000(springx),995(docker) context=un...
```

可以看出，在宿主机中，当前登录的用户的会话是包含 docker 组的（groups = 1000，995，宿主机的 docker 组 id 正好为 995）。

相反的是，在容器内部，即使修改了用户组，登录会话的组还是 docker-compose 配置的 995（groups），所以怎么试都没有权限。

找到问题的根源：**有没有权限，与登录会话有关，需要拥有 docker 组的权限才可以。**

既然知道了问题的根源，那如何让登录会话拥有 docker 组权限呢？

**临时测试方法（`newgrp`）**

想要修改登录会话所属组，就要修改登录身份。也就是说，用户以哪个组的身份登录，就拥有对应组的权限。

我们可以通过 `newgrp` 临时切换到 docker 组，来测试 jenkins 用户是否有权限使用 docker：

```SH
$ newgrp docker
$ id
uid=1000(jenkins) gid=995 groups=998
$ docker version
# ... OK！
```

从上面的切换方式可以看出，jenkins 已经有了权限，但是重新登录后，又会回到原来的组，而且仅仅是当前切换后的会话才有权限，在项目中使用 docker 还是没有权限的。

**永久解决**

既然修改登录的组，就能获取权限，那该如何彻底解决呢？

前面提到，docker-compose 配置了用户 id 和组 id，这将会使用户以该配置的运行，**所以，我们只需要把组 id 指定为和宿主机一样的 docker 组 id 即可**。

从最前面给出的 docker-compose 配置文件，可以看出给 `user` 的配置是 `1000:995`，1000 是宿主机的用户，Jenkins 是该用户部署的，默认用户是 jenkins（1000），配置一样的 id，可以避免映射处理的文件的读写权限问题；995 是宿主机 docker 的组 id，这样容器内部的用户就会以 `1000：955` 的身份登录。

```SH
# 宿主机的 docker 组 id
$ cat /etc/group | grep docker
docker:x:995:springx
```

**小结**

对于容器内的 Jenkins，想要使用宿主机的 docker，只需要在 docker-compose.yaml 中配置 user，指定宿主机 docker 组 id 即可，并不需要在容器内创建 docker 组（实际 Jenkins 容器内并不存在 995 的组，但这并不影响）

```YAML
version: '3.9'
services:
  jenkins:
    image: jenkins/jenkins:jdk17
    container_name: jenkins
    # 指定用户 uid:gid（`用户id`:`docker组id`，跟文件权限有关）
    # 只要 995 这个值与宿主机的 docker 组 id（使用 `cat /etc/group | grep docker` 查看）一致即可！
    user: 1000:995
```

### 2.2 映射目录权限问题

映射到宿主机的目录和文件，同样会遇到权限问题，这个与上面 docker 权限问题相比简单至极。

要么，开放映射目录（`.jenkins_home`）的所有权限。（这样似乎不太好？反正我感觉不好，但我没证据~）

```SH
$ chmod 777 .jenkins_home
```

要么，切换映射目录的归属。例如，我使用普通用户 springx（1000:1000）部署 Jenkins，目录所有者就是 1000，刚好 Jenkins 容器内部默认用户 jenkins 也是 1000:1000，所以，只要用 springx 用户先创建 `.jenkins_home`，然后配置 docker-compose 指定 `user: 1000:gid` 即可。

```SH
[springx@fun ~]$ mkdir .jenkins_home
# 如果使用其他用户创建，需要修改权限
[springx@fun ~]$ chwon 1000:1000 .jenkins_home
```

如果没有提前创建 `.jenkins_home`，它的默认所有者是 root，此时如果使用普通用户部署（就如上诉使用 1000 用户部署），容器内部的用户是没有权限的：

```SH
$ docker-compose up
[+] Running 2/1
 ✔ Network jenkins2_default  Created          
 ✔ Container jenkins2           Created          
Attaching to jenkins2
jenkins2  | touch: cannot touch '/var/jenkins_home/copy_reference_file.log': Permission denied
jenkins2  | Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?
jenkins2 exited with code 0
```

- 这里 `/var/jenkins_home` 映射到了 `.jenkins_home`
- 如果配置了重启策略 `restart: unless-stopped`，只需要修改 `chown springx:springx .jenkins_home`，然后等待重启即可。
- 如果执行 `docker-compose` 的命令是我的普通用户 springx，那么 `.jenkins_home` 是会有权限的，只不过我没有把 springx 用户加入 docker 组，而是用 sudo 执行的

**小结**

如果使用 root 用户部署，映射目录不会有权限问题；如果使用普通用户部署，需要先创建映射目录，并配置相应的权限，否则自动创建的映射目录，属于执行执行创建命令的用户所有，如果用户 id 不一致，会出现无法写入的问题（我用的 sudo，则创建的文件的所有者都是 root）。

## 总结

使用 docker-compose 部署 Jenkins，主要涉及几个权限问题，其中主要的是 docker 权限和映射目录的权限：

docker 权限可以使用 root 部署，或者修改 `docker.sock` 权限（不建议）；也可以通过配置，把容器内部的用户，添加到和宿主机 docker 组一样的 id 组即可（`user: 1000:995`）。

映射目录权限，可以先手动创建，然后分配给相应的用户，并且 docker-compose 配置对应的用户 id 即可；也可以创建后，再修改权限，等待容器自动重启即可。
