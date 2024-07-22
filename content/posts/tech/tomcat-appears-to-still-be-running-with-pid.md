---
title: "Tomcat 停止后无法启动，“Tomcat appears to still be running with PID ...”"
date: 2023-09-04T22:44:03+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "tomcat", "FAQ"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Tomcat 无法启动，报似乎仍在运行中，是因为程序记住了正在运行的 PID，异常终止导致没有正常清空该 PID，再次启动时检测到该 PID，故报 
Tomcat 可能已经正在运行中..."
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
## 问题是什么呢？

在停止 Tomcat 后，无法再次启动。

尝试使用 `catalina.sh` 启动时出现如下警告：

```SH
[R@K tomcat/bin]$ ./catalina.sh start
Using CATALINA_BASE:   /app/tomcat
Using CATALINA_HOME:   /app/tomcat
Using CATALINA_TMPDIR: /app/tomcat/temp
Using JRE_HOME:        /app/j2sdk1.8.0/jre
Using CLASSPATH:       /app/tomcat/bin/bootstrap.jar:/app/tomcat/bin/tomcat-juli.jar
Using CATALINA_OPTS:    -javaagent:/app/skywalking/skywalking-agent/skywalking-agent.jar -Dskywalking.agent.service_name=springx-web  -Dskywalking.collector.backend_service=192.168.1.122:11800
Using CATALINA_PID:    /app/tomcat/bin/CATALINA_PID
Existing PID file found during start.
Tomcat appears to still be running with PID 22832. Start aborted. # 这里说 Tomcat 已经运行了，PID 为 22832
If the following process is not a Tomcat process, remove the PID file and try again:
UID        PID  PPID  C STIME TTY          TIME CMD               # 可是这里并没有任何进程
```

输出的日志显示，PID 已存在。根据部分 Linux 应用的习惯可知，Tomcat 启动后，会将其 PID 记录在一个文件中，再次启动时会主动检测该文件，以判断是否重复启动。

日志还输出了 `ps` 命令的结果，可见并没有 PID 为 22832 的进程。

而通过 `ps -ef | grep java` 也没有找到任何进程，明明 Tomcat 已经停止了，为什么说已经在运行了呢？

## 寻找原因

猜测原因是，Tomcat 停止时由于 JDBC 等其他线程未正常停止，导致 Tomcat 没有正常停止，在二次执行 `kill -9` 后，出现意外。

> 这里为什么会使用 `kill -9` 呢？年代久远，无从可考。原脚本是使用 `catalina.sh stop`，但却在 sleep 几秒之后，使用 `kill -9` 杀死 java 相关进程。
这里大胆猜测，前辈在停止 Tomcat，经常未能正常停止，所以选择几秒之后直接强杀。

## 解决问题

在 Tomcat 的 bin 目录下，有一个 `CATALINA_PID?` 文件，它就是记录 PID 的文件（上述日志有体现）。

```SH
[R@K tomcat/bin]$ ls
catalina64.bat              catalina.bat                     ciphers.bat 
catalina64.sh               CATALINA_PID?  # ←就是这个文件     # ...
catalina64.sh.default       catalina.sh                
catalinabak.sh              catalina-tasks.xml   
[R@K tomcat/bin]$
[R@K tomcat/bin]$ cat CATALINA_PID^M
22832
[R@K tomcat/bin]$ rm CATALINA_PID^M
```

查看该文件的内容，确实是日志报错的 PID。

把该文件删除，可以正常启动 Tomcat 了。并且可以看到该文件被重新生成，并附上了新的 Tomcat PID：

```SH
[R@K tomcat/bin]$ cat CATALINA_PID^M
1369
```

## 碎碎念

会出现以上问题，主要是项目没有、并且不能优雅的停止，导致用上了强杀的手段。

这个项目的可考记录，可以追溯到 0 几年，很多代码没人敢动，旧代码单文件动不动就是几千行，最近头发掉了不少。
