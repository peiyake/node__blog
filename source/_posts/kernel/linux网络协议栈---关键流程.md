---
title: Linux网络协议栈--关键流程
type: kernel
description: linux内核实现了TCP/IP协议栈，整个实现机制扩展性非常强。使用者也能非常方便的添加自己的协议栈。
date: 2020-09-25 09:20:00
---

自己做了一些细节整理，简单画了个蓝图。 

另外，关于网络协议栈这一块，网络上很多人已经有了总结，实在不想重复造轮子，这里再收藏一些总结比较好的博客，记录一下。


## 网络协议栈知识收藏连接

* [很详细的协议栈总结](https://www.cnblogs.com/sammyliu/p/5225623.html)


## 关键流程蓝图

![关键流程](/images/inet.png)

