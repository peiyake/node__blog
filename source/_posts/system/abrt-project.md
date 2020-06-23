---
title: ABRT 是什么？
type: system
description: ABRT 是一个工具集，用于检测和报告应用程序的crash信息；它主要的目的是协助处理问题并找到解决方案
date: 2020-06-22 19:10:42
---

# ABRT 是什么？

>ABRT is a set of tools to help users detect and report application crashes. Its main purpose is to ease the process of reporting an issue and finding a solution.

>ABRT 是一个工具集，用于检测和报告应用程序的crash信息；它主要的目的是协助处理问题并找到解决方案

## 测试环境

* CentOS-7.5.1804
* abrt version: 2.1.11

## 程序介绍

|说明| 程序名称 | 功能 | rpm包 |
|------|---------|-------|-------|
| 主服务器进程| abrtd| ABRT工具集主进程，守护进程，用来收集crash信息 |abrt-2.1.11-50.el7.centos.x86_64 |
| core-dump插件|abrt-hook-ccpp | abrt core-dump 信息收集插件，收集完成会发送给abrtd| abrt-addon-ccpp-2.1.11-50.el7.centos.x86_64|
| oops插件|abrt-dump-oops|abrt内核oops监控插件|abrt-addon-kerneloops-2.1.11-50.el7.centos.x86_64|
| kernel-panic插件|abrt-harvest-vmcore|abrt内核panic监控插件 |abrt-addon-vmcore-2.1.11-50.el7.centos.x86_64|

## Linux程序有哪些崩溃行为？

#### 1 core-dump

用户态应用程序主动产生。通常是用户态应用程序产生 **段错误** 时，通常会生产core文件。一般都是访问了不可访问的内存。

#### 2 oops

Linux内存管理系统如果发现系统内存即将耗尽（通常会有一个阈值），调度系统会根据一定的算法，选择一个应用程序把它杀掉释放内存，来缓解系统的内存压力。
这种情况，被kill的进程可能是无辜的，并非进程自己主动犯错的。

#### 3 kernel-panic

内核态程序产生。  这个跟用户态差不多，如果内核模块访问了不可访问的内存，就会产生 **kernel-panic**。 这种情况通常会造成系统重启或者宕机。

## abrt

ABRT工具集对于以上三种系统错误都可以进行收集分析，每种崩溃行为的分析方法，请看其它文章。



