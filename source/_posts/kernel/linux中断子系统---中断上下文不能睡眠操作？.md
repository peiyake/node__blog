---
title: linux中断子系统---为什么中断上下文不能执行可能睡眠操作的操作？
type: kernel
description: 内核运行在中断上下文时，内核无法进行调度，所以一旦存在睡眠，内核就不知道该干啥了，调度系统傻了，就panic了
date: 2020-09-16 16:57:54
---

内核运行在中断上下文时，内核无法进行调度，所以一旦存在睡眠，内核就不知道该干啥了，调度系统傻了，就panic了。