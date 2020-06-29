---
title: git clone/add/diff
type: shell
description: Git命令 clone、add、diff
date: 2020-06-22 19:10:42
---

本文使用github上创建的示例仓库进行练习 `git@github.com:peiyake/git_cmd_test.git`

## [git clone](https://git-scm.com/docs/git-clone/zh_HANS-CN)

1. 普通克隆

    git clone git@github.com:peiyake/git_cmd_test.git

2. 克隆到指定目录
   
    git clone git@github.com:peiyake/git_cmd_test.git  ~/piak/my_git_cmd_test

3. 克隆远端指定分支，使用 `-b <branch name>`

    git clone git@github.com:peiyake/git_cmd_test.git -b b1


## [git add](https://git-scm.com/docs/git-add/zh_HANS-CN)

>在对工作树进行任何更改之后，以及在运行commit命令之前，必须使用`add`命令将所有新文件或修改过的文件添加到索引中

1. 参数

    * `-n | --dry-run`，实际上不添加文件，仅展示文件是否存在或是否忽略。
    * `-A`，如果在使用 -A 选项时没有 <指定路径>，则整个工作树中的所有文件都将更新（旧版本的 Git 会限制当前目录及其子目录的更新）。
    * `--chmod=(+|-)x `，重写添加文件的可执行位。可执行位仅在索引中更改，磁盘上的文件保持不变。

2. 示例
   
```
[peiyake@localhost git_cmd_test]$ touch 4.c
[peiyake@localhost git_cmd_test]$ git status
# On branch b1
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#       4.c
nothing added to commit but untracked files present (use "git add" to track)
[peiyake@localhost git_cmd_test]$ git add -A
[peiyake@localhost git_cmd_test]$ git status
# On branch b1
# Changes to be committed:
#   (use "git reset HEAD <file>..." to unstage)
#
#       new file:   4.c
#
[peiyake@localhost git_cmd_test]$
```

## [git diff](https://git-scm.com/docs/git-diff)

1. `git diff`，显示未被[git add](#git-addhttpsgit-scmcomdocsgit-addzhhans-cn)过的文件的变更，（注意如果是新增的文件，没有commit过的话，不会显示出来）。
2. `git diff --cached`，显示已经被[git add](#git-addhttpsgit-scmcomdocsgit-addzhhans-cn)但是还没有`commit`的变更。
3. `git diff HEAD`，显示以上所有变更
4. `git diff <commit id>`，比较 **<commit id>** 和 **<HEAD>** 之间的差异
5. `git diff <commit id1> <commit id2>`，比较 **<commit id1>** 和 **<commit id2>**之间的差异
5. `git diff <commit id1> <commit id2> --stat`，只显示 **<commit id1>** 和 **<commit id2>**之间存在差异的文件列表

