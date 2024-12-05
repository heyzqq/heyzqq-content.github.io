---
title: "Git Worktree Cheatsheet (Worktree 速查表)"
date: 2024-11-28T21:51:17+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "git"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "本文档提供了 Git Worktree 的简洁指南，包括如何管理和操作多个工作区以提高开发效率。"
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

docs: [https://git-scm.com/docs/git-worktree](https://git-scm.com/docs/git-worktree)

## 01 基本用法

```sh
git worktree add <path> [<branch>]            # 在指定路径创建一个新的工作区，可选地检出指定分支
git worktree list                             # 列出所有工作区及其状态
git worktree prune                            # 清理无效的工作区引用
```

## 02 工作区管理

```sh
git worktree remove <path>                     # 移除指定的工作区（需要先清理工作区内的未提交更改）
git worktree move <source> <destination>       # 移动或重命名现有工作区
git worktree lock <path>                       # 锁定工作区，防止误操作
git worktree unlock <path>                     # 解锁工作区
```

## 03 分支操作

```sh
git worktree add -b <new-branch> <path>         # 基于当前分支创建新的分支并检出到新工作区
git checkout -b <new-branch> --work-tree=<path> # 在现有工作区中创建并切换到新分支
git branch -D <branch> --work-tree=<path>       # 删除指定工作区中的分支（需确保工作区干净）
```

## 04 高级使用

```sh
git worktree add --detach <path>               # 创建一个分离头指针的工作区
git worktree add --no-checkout <path>          # 创建工作区但不检出文件
git worktree add -f <path>                     # 强制创建工作区，即使目标目录非空
git worktree add --track <path> <remote>/<branch> # 克隆远程分支并跟踪
```

## 05 最佳实践

1. 使用有意义的路径名称来标识每个工作区的目的。
2. 定期运行 `git worktree prune` 来清除不再存在的工作区引用。
3. 在多个工作区之间切换时，确保没有未提交的更改。
4. 对于长期存在的特性分支，可以考虑使用独立的工作区。
5. 使用 `--detach` 创建分离头指针的工作区来进行实验性开发，完成后可以通过创建新分支来保存更改。
6. 在团队环境中使用工作区时，确保团队成员了解工作区的存在，并在共享仓库前解决任何冲突。
7. 如果工作区位于不同的物理位置，使用 `--no-checkout` 选项以避免不必要的文件复制。
8. 当需要同时处理多个特性或任务时，利用工作区提高效率。
9. 对于大型项目，考虑将不同模块或组件放在不同的工作区中开发。
10. 使用 `git worktree lock` 和 `git worktree unlock` 来保护重要工作区免受意外操作的影响。

示例：从当前分支检出新的 worktree，修复 bug，提交，然后清除工作区

```SH
# 1. 示例：从当前分支检出新的 worktree，以下命令会在上级目录创建和当前分支一样的拷贝
> git worktree add ../project_name-hotfix
> git worktree list
D:/Work/project_name         128056e [feature/test]
D:/Work/project_name-hotfix  128056e [feature/test]
> ls -l ../
.
..
project_name-hotfix/
project_name/
# 2. 修改并提交
> git add . && git commit -m "fix"
> git push
# 删除无用 worktree
> cd ../project_name
> git worktree remove ../project_name-hotfix
> git worktree list
D:/Work/project_name         128056e [feature/test]
```

**这里需要注意**：使用 `git worktree add -b new-branch ../project_name` 时，如果直接推送的话，并不是推送 `new-branch` 分支，而是相当于在原分支提交并推送。**修复 bug 时候要注意，容易不小心直接提交到 master 上 ......**

在创建新的工作区时，可以看到默认会把新的工作区/新的分支关联到源分支：

```sh
# git worktree add -b <new-branch> ../project_name <src-branch>
> git worktree add -b f-new ../proj-ww origin/f-test
Preparing worktree (new branch 'f-new')
branch 'f-new' set up to track 'origin/f-test'.  # 这里可以看到，关联了远程分支
Updating files: 100% (1024/1024), done.
HEAD is now at 1314520a commit_info
```

如果要提交新的分支，需要从指定的 commit-id 切出：

```sh
# git worktree add -b <new-branch> ../project_name <commit-id>
> git worktree add -b f-new  ../proj-ww 195d5bbf
Preparing worktree (new branch 'f-new')
Updating files: 100% (1024/1024), done.
HEAD is now at 195d5bbf fix sth.
```

## 注意事项

- 确保 Git 版本支持 `git worktree` 功能（Git 2.5+）。
- 当工作区不在同一文件系统时，可能会遇到性能问题。
- 不要直接删除工作区文件夹，应使用 `git worktree remove` 或 `git worktree prune`。
- 分离头指针的工作区容易导致“丢失”提交，使用时需小心。
- 避免在同一个工作区中频繁切换分支，这可能导致工作区污染。
