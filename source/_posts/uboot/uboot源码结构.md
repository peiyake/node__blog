---
title: uboot源码结构
type: uboot
description: uboot是和硬件平台紧密相关的，了解其代码结构有助于uboot移植和修改。
date: 2020-09-03 09:33:42
---

工作需要，本人近段时间开始了解uboot，开发任务完成了一阶段。回过头来想把最近有关uboot的理解整理一下，方便自己今后查阅。


## uboot代码目录结构
|目录名称||说明|
|------|----|-----|
|/arch		|	|不同CPU架构相关的内容，例如arm、mips、x86...|
|/api		|	|Machine/arch independent API for external apps|
|/board		|	|板级相关代码|
|/common	|	|Misc architecture independent functions|
|/configs	|	|Board default configuration files|
|/disk		|	|Code for disk drive partition handling|
|/doc		|	|Documentation (don't expect too much),帮助文档|
|/drivers	|	|Commonly used device drivers,总线、硬件驱动|
|/dts		|	|Contains Makefile for building internal U-Boot fdt.uboot中fdt模块使用的设备树|
|/examples	|	|Example code for standalone applications, etc.|
|/fs		|	|Filesystem code (cramfs, ext2, jffs2, etc.)|
|/include	|	|Header Files,头文件。 目录结构也会根据CPU架构来区分，另外板级的头文件在include/configs目录下|
|/lib		|	|Library routines generic to all architectures,公共代码库|
|/Licenses	|	|Various license files|
|/net		|	|Networking code|
|/post		|	|Power On Self Test|
|/scripts	|	|Various build scripts and Makefiles|
|/test		|	|Various unit test files|
|/tools		|	|Tools to build S-Record or U-Boot images, etc.|


uboot的代码结构根据功能进行了模块划分，大概的层次结构如下图所示：

![uboot代码组织结构图](/images/uboot_code.png)


下面呢，对上面的目录一一进行介绍


### /arch

```
arch
├── arc
├── arm
├── avr32
├── mips
├── powerpc
├── ...
└── x86
```

1. 不同CPU架构使用的指令集不同。简单的例如：x86使用复杂指令集、arm使用精简指令集
2. 架构不同，导致同一段C代码编译出来的汇编程序就不一样。
3. 我们知道C程序里面可以嵌入汇编，那么一个函数里面嵌入的汇编语句可能就要根据不同的cpu架构使用不同的汇编代码

另外，相同架构的CPU，由于在不断的发展过程中，会产生不同系列的架构子集，以arm为例：arm架构的子集有armv7/armv8/arm11...

由于以上原因，在uboot中还要对这些子集进行区分，我们看一下uboot中arm/arm 如何进行目录归类

```
arch/arm/cpu
├── arm11
├── arm1136
├── arm1176
├── arm720t
├── arm920t
├── arm926ejs
├── arm946es
├── armv7
├── armv7m
├── armv8
├── pxa
└── sa1100
```

搞清楚以上的关系，你就知道当你开发一款硬件时，怎么在uboot代码中找到真正使用的代码。 例如：我这边使用高通ipq6018的cpu，架构是armv7，那么当我找启动流程
的`start.S`时，我就直接到`arch/arm/cpu/armv7`目录下去找就可以了，一定不会错。

**arch目录下的东西一般不需要修改的，只要你的硬件cpu架构在这里能找到，那么所要做的工作就是找到对应目录和对应文件，修改、添加自己的东西就可以了**

### /api

统一API模型，该目录下的api统一使用 `syscall`来调用

### /board

何谓板级？ 我的理解是PCB。

虽然CPU的架构不同，但是通常CPU都会设计专用功能的接口，例如：I2C/SPI/USB... 这些总线协议在不同CPU上的代码实现，必定存在公共和不同的部分。板级代码呢就是在这个层面上
进行划分的。 例如，我这里使用高通ipq6018平台，板级代码目录结构如下：

```
board/qca/arm
├── common
│   ├── board_init.c
│   ├── cmd_blowsecfuse.c
│   ├── cmd_bootqca.c
│   ├── cmd_exectzt.c
│   ├── cmd_runmulticore.c
│   ├── cmd_sec_auth.c
│   ├── cmd_tzt.c
│   ├── crashdump.c
│   ├── env.c
│   ├── ethaddr.c
│   ├── fdt_fixup.c
│   ├── fdt_info.c
│   ├── fdt_info.h
│   └── Makefile
├── ipq6018
│   ├── clock.c
│   ├── ipq6018.c
│   ├── ipq6018.h
│   └── Makefile
└── ipq807x
    ├── clock.c
    ├── ipq807x.c
    ├── ipq807x.h
    └── Makefile
```

### /configs

uboot编译时，必须先执行 make xxx_defconfig，那么这个 `xxx_defconfig` 按照惯例就存放在这个目录下

### /common

common目录下的东西已经基本上做到硬件无关了，我们对UBOOT进行开发，大部分都是需要修改该目录下的内容。


**另外还有一些其它目录，自己也不是很清楚，目前的工作都没有涉及到，所以今后有时间了再进行深入研究**