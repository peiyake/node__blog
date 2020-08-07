---
title: OpenWrt之package详解一（组成部分）
type: openwrt
description: OpenWrt中package的组成部分及其作用详解
date: 2020-07-14 16:18:42
---

构造OpenWrt Package，首先要了解Packege的组成部分和各个组成部分的功能和作用。

## Package 组成部分

根据前文[openwrt的feed机制](https://peiyake.com/2020/07/09/openwrt/openwrt%E7%9A%84feed%E6%9C%BA%E5%88%B6/)，我们知道OpenWrt实际上就是由一系列包构成的一个系统，只不过这些包涵盖了
系统所需的内核、文件系统、应用程序等。我们当然也可以构建自己的Packege来安装到系统中以实现我们自己的应用。

一个OpenWrt Package 用一个文件夹来描述，这个文件夹包括以下几个组成部分：

* `package/Makefile` （必须）
* `package/patches/` （可选）
* `package/files/` （可选）
* `package/src/` （可选）


## Makefile

An OpenWrt source package Makefile contains a series of header variable assignments, action recipes and one or multiple OpenWrt specific signature footer lines identifying it as OpenWrt specific package Makefile

一个OpenWrt Package的Makefile包含几个部分：变量定义，动作方法和一个或多个Openwrt 特定方法来标识这是一个OpenWrt特定Makefile

## package/files

Static files accompanying a source package, such as OpenWrt specific init scripts or configuration files, must be placed inside a directory called files, residing within the same subdirectory as the Makefile. There are no strict rules on how such static files are to be named and organized within the files directory but by convention, the extension .conf is used for OpenWrt UCI configration files and the extension .init is used to denote OpenWrt specific init scripts. 

对于`files`文件夹的使用只是一个惯例，并没有严格的规定。1. 使用`files`这个名称只是公共约定，事实上你可以任意指定，只不过在Makefile中也要使用你定义的名称；2. `files`文件夹应该和Makefile在同一级目录；3. `files`文件夹应该放置一些静态文件，例如：inin scripts 或者 配置文件；4. 按照约定，扩展名 `.init`的文件作为init scripts，扩展名`.conf`的文件作为Openwrt LUCI的配置文件.

## package/patches

The patches directory must be placed in the same parent directory as the Makefile file and may only contain patch files used to modify the source code being packaged.Patch files must be in unified diff format and carry the extension .patch. The file names must also carry a numerical prefix to denote the order in which the patch files must be applied. Patch file names should be short and concise and avoid special characters such as white spaces, special characters and so on. 

同样的，`patches` 的使用也是一个惯例，并没有严格规定。1. patches应该和Makefile放置在同一级目录；2. patches文件名字必须以数字开头；3. patches的名字应该简明、扼要，不应该包括特殊字符。OpenWrt中对于patches的使用和管理有专门的工具，请参考 [Quilt](https://openwrt.org/docs/guide-developer/build-system/use-patches-with-buildsystem)

## package/src

有些项目可能不想从远端获取package的源码，那么可以直接把源码维护在src目录下。另外，src目录也可以作为package的扩展代码存放目录。总之呢，src的使用也没有严格的规定。

