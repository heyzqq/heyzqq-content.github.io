---
title: "Docker 之 Linux CGroup"
date: 2019-02-28T22:20:34+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "linux"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "本文介绍了 Linux CGroup 的基本概念和相关资源子系统，并详细讲解了如何使用 CPU 限制。首先通过在 cgroup 对应的子资源目录下创建目录并往对应文件写入分配值来限制 CPU 占用率，然后运行测试程序并将其 PID 写入相应文件，最后成功限制 CPU 占用率。本文还介绍了如何移出限制，即结束进程或将 pid 写入根 cgroup 的 tasks 文件。"
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
## 基本概念

Linux CGroup 全称 Linux Control Group， 是 Linux 内核的一个功能，用来限制，控制与分离一个进程组群的资源（如 CPU、内存、磁盘输入输出等）。  

主要提供了如下功能：  

- Resource limitation: 限制资源使用，比如内存使用上限以及文件系统的缓存限制。
- Priority control : 优先级控制，比如：CPU 利用和磁盘 IO 吞吐。
- Accounting: 一些审计或一些统计，主要目的是为了计费。
- Control: 挂起进程，恢复执行进程。  

一些基本的资源子系统：

- Block IO（blkio)：限制块设备（磁盘、SSD、USB 等）的 IO 速率
- CPU Set(cpuset)：限制任务能运行在哪些 CPU 核上
- CPU Accounting(cpuacct)：生成 cgroup 中任务使用 CPU 的报告
- CPU (CPU)：限制调度器分配的 CPU 时间
- Devices (devices)：允许或者拒绝 cgroup 中任务对设备的访问
- Freezer (freezer)：挂起或者重启 cgroup 中的任务
- Memory (memory)：限制 cgroup 中任务使用内存的量，并生成任务当前内存的使用情况报告
- Network Classifier(net_cls)：为 cgroup 中的报文设置上特定的 classid 标志，这样 tc 等工具就能根据标记对网络进行配置
- Network Priority (net_prio)：对每个网络接口设置报文的优先级
- perf_event：识别任务的 cgroup 成员，可以用来做性能分析


## CPU 限制

### 创建 cgroup

首先，直接在 cgroup 对应的子资源目录下, 用 `mkdir` 创建一个目录，就会自动包含必要文件

```sh
[springx.fun@ /sys/fs/cgroup/cpu]$ mkdir test
[springx.fun@ /sys/fs/cgroup/cpu]$ ls test/
cgroup.clone_children  cpuacct.usage         cpuacct.usage_percpu_sys   cpuacct.usage_user  cpu.shares         tasks
cgroup.procs           cpuacct.usage_all     cpuacct.usage_percpu_user  cpu.cfs_period_us   cpu.stat
cpuacct.stat           cpuacct.usage_percpu  cpuacct.usage_sys          cpu.cfs_quota_us    notify_on_release
```

然后，限制 CPU 占用率，其实就是往对应文件写入分配值。`cpu.cfs_quota_us` 是最大 CPU 时间片（单位微妙，us）

- `-1`：表示无限制
- `20000`：在每个 100ms 的调度周期中，该 Cgroup 下的任务只能占用 CPU 时间的 20ms，超出后会被挂起，等待下一个周期的调度

```sh
[springx.fun@ /sys/fs/cgroup/cpu]$ cat test/cpu.cfs_quota_us
-1
[springx.fun@ /sys/fs/cgroup/cpu]$ echo 20000 > test/cpu.cfs_quota_us
[springx.fun@ /sys/fs/cgroup/cpu]$ cat test/cpu.cfs_quota_us
20000
```

### 运行测试

首先, 创建一个无限循环的程序:  

```c
int main(void)
{
    int i = 0;
    for(;;)
        i++;
    return 0;
}
```

然后, 直接运行这个程序，当前是没有任何限制的，可以看到 CPU 已经占用到 100% 了:  

```
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND            
29637 zqq       20   0    4220    632    560 R 100.0  0.0   0:07.58 a.out  
```

接着, 将 a.out 的 PID 写入 `/sys/fs/cgroup/cpu/test/tasks`  

```
[springx.fun@ /sys/fs/cgroup/cpu]$ echo 29637 >> test/tasks
```

现在，可以看到 CPU 徘徊在 20% 左右

```
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND            
29637 zqq       20   0    4220    632    560 R  19.9  0.0  11:57.64 a.out 
```

最后，结束 a.out 进程后, test 目录下的 tasks 的 PID 就会被清空.  

### 移除限制

有两个方法移出限制:  

1. 最简单的, 结束进程即可.

2. 比较友好的方法, 就是将把 pid 写入到根 cgroup 的 tasks 文件即可(`/sys/fs/cgroup/cpu/tasks`).   
   因为每个进程都属于且只属于一个 cgroup, 加入到新的 cgroup 后, 原有关系也就解除了.  
   ```shell  
   [springx.fun@ /sys/fs/cgroup/cpu]$ echo 29637 >> tasks
   ```

3. 删除创建的 cgroup, 需要先将 tasks 都移除:
   ```shell
   [springx.fun@ /sys/fs/cgroup/cpu]$ rmdir test
   ```

---

## REFERENCES

[1] 左耳朵耗子. DOCKER基础技术：LINUX CGROUP[EB/OL]. https://coolshell.cn/articles/17049.html, 2015-04-17.  
[2] 神仙的仙居. linux cgroups 概述[EB/OL]. https://xiezhenye.com/2013/10/linux-cgroups-%E6%A6%82%E8%BF%B0.html, 2013-10-23.  
