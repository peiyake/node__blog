---
title: Linux内核驱动层架构
type: kernel
description: linux内核中对于硬件相关的部分都构建了框架，驱动工程师和BSP工程师分属框架的两端，彼此不需要关心对方的实现细节，只需要通过框架API来联系。这种做法让擅长硬件的BSP工程师更能专注于硬件接口，驱动工程师则不需要关注太多硬件层面的东西，大大减轻了驱动工程师的工作量。
date: 2020-09-15 10:00:00
---

## Linux内核驱动层框架图

![驱动层框架图](/images/driver_framework.png)

* 驱动和应用程序接口：read/write/ioctl
* 驱动和BSP接口：GPIO子系统为例**linux/gpio.h**/**linux/gpio/consumer.h**
* 设备树：应该说BSP工程师来搭建设备树框架，驱动工程师对设备树的修改只是简单的一小部分


具体细节可参考GPIO子系统介绍[https://peiyake.com//2020/09/15/kernel/内核GPIO系统理解/](https://peiyake.com//2020/09/15/kernel/内核GPIO系统理解/)

