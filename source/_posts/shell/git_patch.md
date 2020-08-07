---
title: git 制作patch，打patch
type: shell
description: 使用git可以制作标准的patch，对于项目中代码管理非常方便。主要用到format-patch/apply
date: 2020-08-07 19:10:42
---

使用git可以制作标准的patch，对于项目中代码管理非常方便。


### 制作patch

`git format-patch` 会把每次提交都单独生成一个patch文件。

* `git format-patch` 默认是从**最新**提交ID为起点
* `git format-patch --root` 默认是从**首次*8提交ID为起点

#### 举例

最近一次提交，制作patch

    git format-patch HEAD^

某两次提交之间，制作patch

    git format-patch <commit id 1> <commit id 2>

单独某次提交，制作patch

    git format-patch -1 <commit id>

指定某次提交为参照点，制作该参照点之前的N次提交的patch

    git format-patch -N <commit id>

制作从首次提交到 某个指定提交点之间的所有patch

    git format-patch --root <commit id>

制作所有提交的patch

    git format-patch --root HEAD


### 打patch

检查patch是否可用

    git apply --check <patch file>

打单个patch

    git apply  <patch file>

打多个patch

    git apply  <patch dir>/*.patch 

打入patch后，自动根据patch的commit信息自动提交

    git am  *.patch

#### 打patch有冲突怎么解决？

网上有的同志说，用 `git am --continue、git am --skip、git am --abort`。但是我觉得，如果存在冲突 最好就不要再继续打patch了，而是仔细看看哪里存在冲突，然后使用可视化的对比编辑器，来解决冲突问题。因为使用以上命令，一边打patch一边解决冲突，如果存在大量冲突，一个一个看的话很容易出错的。



    