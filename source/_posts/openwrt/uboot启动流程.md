---
title: u-boot启动流程
type: openwrt
description: 本文讲解了解uboot的启动流程，才能更透彻理解嵌入式系统的精髓。
date: 2020-07-22 09:33:42
---

之前的大部分工作都是在X86环境下的开发，设计用户态、内核态，最近工作调整，开始了OpenWrt下的嵌入式设备开发，更接近底层一些。兴奋之余，发现底层的东西还真不好整，
搞得有些头大，但这就是工作，没有挑战怎么成长呢？

目前的工作需要在uboot上修改一些东西，通过几天的了解，终于有了写眉目，这里总结一下方便自己查阅。另外，在自己了解的过程中拜读了一个大牛的博客，才得以进展迅速，这里
附上博客链接：[[uboot] （第五章）uboot流程——uboot启动流程](https://blog.csdn.net/ooonebook/article/details/53070065)。这位大牛讲解的非常细致，本文结合自己的
项目和理解再自己总结一下，所谓“好记性不如烂笔头”，看再多，只有自己消化了才真正属于自己的。

## 目标平台信息

* QCA-SDK之OpenWrt
* uboot-2016
* qca60xx（32bit）
* armv7

## 启动过程设计文件流

* arch/arm/cpu/u-boot.lds
  * `ENTRY(_start)`
* arch/arm/lib/vectors.S
  * `_start:`
  * `b reset`
* arch/arm/cpu/armv7/start.S
  * `reset:`
  * `b	save_boot_params`
  * `b	save_boot_params_ret`
    * `bl	cpu_init_cp15`
    * `bl	cpu_init_crit`
      * `b	lowlevel_init`
  * `bl	_main`
* arch/arm/lib/ct0.S
  * `bl	board_init_f_mem`
  * `bl	board_init_f`
    * `common/board_f.c`
  * `ldr	pc, =board_init_r`
    * `common/board_r.c`
      * 