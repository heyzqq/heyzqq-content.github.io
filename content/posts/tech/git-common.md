---
title: "Git 常用命令"
date: 2023-12-05T21:34:59+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "git"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Git 中储藏功能、恢复功能、标签操作以及分支操作的使用方法和相关命令，包括如何暂存部分文件、取消提交、创建与管理标签、以及分支的创建、切换、删除和合并等常见操作。"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
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

## 01 储藏功能（Stash）

```BASH
git stash list [<log-options>]
git stash push（原 git stash save）
```

### 1.1 git stash 保存部分文件

`git stash`是`git stash push`命令省略写法

`git stash push`指令本身就支持指定文件。

```bash
$ git stash push <file1> <file2> <file3> [file4 ...]
```

```bash
$ git stash push -m "测试" src/java/**/*.java Help.java OK.java
```

### 1.2 git stash -u 保存未跟踪文件

```BASH
$ git stash save 
    [-p|--patch] 
    [-k|--[no-]keep-index] 
    [-q|--quiet]
    [-u|--include-untracked] *
    [-a|--all] 
    [<message>]
```

## 02 恢复功能（Reset）

### 2.1 git reset 取消 Add

误 add 多个文件，只撤销部分文件

```bash
$ git reset HEAD /path/to/dir/or/file
```

```BASH
$ git reset HEAD szy-admin/src/test/com/szy/web/SysUserServiceTest.java
Unstaged changes after reset:
M       szy-analysis/src/main/resources/mapper/analysis/fbt/FbtRequestLogMapper.xml
M       szy-system/src/main/resources/mapper/system/SysUserMapper.xml
```

### 2.2 git reset 取消 commit

```shell
# 取消提交，回复修改的文件
$ git reset --soft HEAD^

# 取消提交，丢弃修改
$ git reset --hard HEAD^

# 回退上上上一个版本 
$ git reset --soft HEAD~3   
```

## 03 标签（Tag）

### 3.1 查询

```SH
# 列出所有
$ git tag

# 搜索符合模式的Tag
$ git tag -l 'v0.1.*''
```

### 3.2 打标签

```SH
# 轻量标签
$ git tag v0.1.2

# 附带说明的标签
$ git tag -a v0.1.2 -m "0.1.2 测试版本"

# 打在指定 commit 上，只需在上面两个命令后加上 commit id
$ git tag v0.1 88772sa00
```

### 3.3 删除标签

```SH
$ git tag -d v0.1.2
```

### 3.4 推送标签

```SH
# 将v0.1.2 Tag提交到git服务器
$ git push origin v0.1.2 

# 将本地所有Tag一次性提交到git服务器
$ git push origin --tags 
```

### 3.5 获取最新 tag

1.  Shell sort 排序
    *   获取当前分支标签：`--merged BRANCH_NAME/commitId`，不指定默认 HDEA（默认当前分支）
    *   Shell 排序：sort
        *   `-t`：指定分隔符
        *   `-n`：依照数值的大小排序
        *   `-r`：以相反的顺序来排序
        *   `-k`：指定需要排序的列

```SH
$ git tag --merged test | sort -t. -n | tail -n 1
```

1.  git 自带 `--srot=<key>` 参数
    *   `version:refname`：tag 被当成版本号排序，同 `<v:refname>`

```SH
$ git tag --sort=version:refname --merged | tail -n 1
```

## 04 分支操作

### 4.1 分支基本操作

```SH
# 查看本地分支
> git branch
# 查看远程分支
> git branch -r

# 创建分支但不切换
> git branch <branch-name>
# 切换到指定分支
> git checkout <branch-name>
# 创建并切换到新分支（以当前所在分支为基础）
> git checkout -b <branch-name>
# git 2.23+
> git switch -c <branch-name>

# 删除本地分支
> git branch -d <branch-name>
# 强制删除本地分支（即使有未合并的更改）
> git branch -D <branch-name>

# 重命名本地分支
> git branch -m <new-branch-name>


# 从指定分支新建分支并切换（或 switch -c）
> git checkout -b <branch-name> a123213cid
Switched to a new branch <branch-name> 
# 注意：这里不能指定分支名称，否则会把新分支关联到该分支，相当于该分支的别名
# 等于是直接在操作原分支，如果指定的是 master，会直接操作 master
> git switch -c hotfix/hello origin/master
branch 'hotfix/hello' set up to track 'origin/master'.
Switched to a new branch 'hotfix/hello'
```

### 4.2 分支合并

1.  合并提交，并生成一个合并提交
    - 不能保持 master 分支干净，但是保持了所有的 commit history，大多数情况下都是不好的，个别情况挺好
    ```
    $ git merge <branch-name>
    ```
2.  压缩合并，多个提交合并成一个提交
    - 以保持 master 分支干净，但是 master 中 author 都是 maintainer，而不是原 owner
    ```
    $ git merge <branch-name> --squash
    # 这时候，修改的文件在当前分支都变成未提交

    # 下一步，提交这些记录，生成一个新的提交（可能要解决冲突）
    $ git commit -m "feat: xxx"
    ```
3.  变基合并，可以尽可能保持 master 分支干净整洁，并且易于识别 author
    - 变基：在要合并的分支 `git rebase -i 要合并进去的目标分支`
    - 合并：在目标分支合并 `git merge 要合并的分支`
    - 变基其实就是找到两个分支共同的祖先，然后在当前分支上合并从共同祖先到现在的所有 commit
    ```
    ce6ecf4 - (HEAD -> master, test) add sth.3 (8 minutes ago) \<zqq> (test commit, master merged)
    82f08db - add sth.2 (29 minutes ago) \<zqq> (test commit)
    3d9a1f4 - add sth. (35 minutes ago) \<zqq>  (test commit)
    f620135 - init (37 minutes ago) \<zqq>
    ```


## References

[1] 魔地主. Git tag 给当前分支打标签. <https://blog.csdn.net/u013802160/article/details/55103186>, 2017-02-14.
