---
title: Linux dump crash工具使用
type: shell
description: 内核 kdump 是一个强大的内核调试工具，你可以这么理解，当 linux内核崩溃时，这个 kdump工具可以将崩溃现场完整的保存下来，也就是内核崩溃时系统的运行状态及各种堆栈信息
date: 2020-06-22 19:10:42
---

内核 **kdump** 是一个强大的内核调试工具，你可以这么理解，当 linux内核崩溃时，这个 **kdump** 工具可以将崩溃现场完整的保存下来，也就是内核崩溃时系统的运行状态及各种堆栈信息。 生成的通常是一个很大的文件，当然这个文件的大小你可以自己设定，不过我们通常都是让内核自己来定。

多亏了这个工具，当我在调试内核模块不知所措时，看到了一丝曙光，下面简单把我这边的使用方法总结记录一下！

## 测试环境

* CentOS-7.5.1804

## dump服务安装

1. 安装**kdump** 服务
```shell
yum install -y kexec-tools          // 安装dump服务软件包
systemctl enable kdump.service      // 设置 kdump 服务开机启动
systemctl status kdump.service      // 查看 kdump 服务当前运行状态
```
2. 开启内核crash

修改系统启动参数，通常是 grub.cfg 文件，在启动参数中添加如下：

    crashkernel=auto

## 内核堆栈调试

例如：我们这里有个内核模块 **l2w.ko** ，该内核模块访问非法内存，产生内核 **panic**，通常这时会在系统的 **/var/crash/** 目录下生成dump
文件，那么但系统重启后，我们就可以通过这个文件来定位内核 **panic** 的原因。

1. **/var/crash/** 目录下内容，如下：

    [root@localhost ~]# ls /var/crash/

    127.0.0.1-2019-09-23-08:44:21

2. 执行如下命令

```shell
[root@localhost ~]# crash  /lib/modules/3.10.0.862.ac/build/vmlinux /var/crash/127.0.0.1-2019-09-23-09\:16\:44/vmcore

crash 7.2.3-8.el7
Copyright (C) 2002-2017  Red Hat, Inc.
Copyright (C) 2004, 2005, 2006, 2010  IBM Corporation
Copyright (C) 1999-2006  Hewlett-Packard Co
Copyright (C) 2005, 2006, 2011, 2012  Fujitsu Limited
Copyright (C) 2006, 2007  VA Linux Systems Japan K.K.
Copyright (C) 2005, 2011  NEC Corporation
Copyright (C) 1999, 2002, 2007  Silicon Graphics, Inc.
Copyright (C) 1999, 2000, 2001, 2002  Mission Critical Linux, Inc.
This program is free software, covered by the GNU General Public License,
and you are welcome to change it and/or distribute copies of it under
certain conditions.  Enter "help copying" to see the conditions.
This program has absolutely no warranty.  Enter "help warranty" for details.
 
GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-unknown-linux-gnu"...

WARNING: kernel relocated [34MB]: patching 82553 gdb minimal_symbol values

      KERNEL: /lib/modules/3.10.0.862.ac/build/vmlinux                 
    DUMPFILE: /var/crash/127.0.0.1-2019-09-23-09:16:44/vmcore  [PARTIAL DUMP]
        CPUS: 8
        DATE: Mon Sep 23 17:16:23 2019
      UPTIME: 00:08:02
LOAD AVERAGE: 0.29, 0.49, 0.33
       TASKS: 277
    NODENAME: localhost
     RELEASE: 3.10.0.862.ac
     VERSION: #1 SMP Mon Aug 26 08:42:15 UTC 2019
     MACHINE: x86_64  (3392 Mhz)
      MEMORY: 4 GB
       PANIC: "BUG: unable to handle kernel NULL pointer dereference at 00000000000000e8"
         PID: 0
     COMMAND: "swapper/3"
        TASK: ffff90bca94ccf10  (1 of 8)  [THREAD_INFO: ffff90bca94e0000]
         CPU: 3
       STATE: TASK_RUNNING (PANIC)

crash> ^C
crash> ^C
crash> bt
PID: 0      TASK: ffff90bca94ccf10  CPU: 3   COMMAND: "swapper/3"
 #0 [ffff90bcafac3188] machine_kexec at ffffffff83260b2a
 #1 [ffff90bcafac31e8] __crash_kexec at ffffffff83313402
 #2 [ffff90bcafac32b8] crash_kexec at ffffffff833134f0
 #3 [ffff90bcafac32d0] oops_end at ffffffff83917768
 #4 [ffff90bcafac32f8] no_context at ffffffff83907088
 #5 [ffff90bcafac3348] __bad_area_nosemaphore at ffffffff8390711f
 #6 [ffff90bcafac3398] bad_area_nosemaphore at ffffffff83907290
 #7 [ffff90bcafac33a8] __do_page_fault at ffffffff8391a720
 #8 [ffff90bcafac3410] do_page_fault at ffffffff8391a915
 #9 [ffff90bcafac3440] page_fault at ffffffff83916768
    [exception RIP: l2w_vdev_hard_xmit_broad_pkt+248]
    RIP: ffffffffc05a3b78  RSP: ffff90bcafac34f0  RFLAGS: 00010216
    RAX: 0000000000000000  RBX: ffff90bc28050a80  RCX: 0000000000000005
    RDX: ffff90bcafc00340  RSI: 00000000000000e0  RDI: ffffffffc05a7480
    RBP: ffff90bcafac35b0   R8: 000000000000000d   R9: 0000000000000000
    R10: ffff90bcafc003a2  R11: ffffffff83efc940  R12: ffff90bca8465800
    R13: ffff90bca8465800  R14: ffffffff83f4be40  R15: ffff90bca8465800
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
#10 [ffff90bcafac35b8] nf_l2w_pre_routing at ffffffffc05a0268 [l2w]
#11 [ffff90bcafac3620] nf_iterate at ffffffff83827408
#12 [ffff90bcafac3660] nf_hook_slow at ffffffff838274f8
#13 [ffff90bcafac3698] br_handle_frame at ffffffffc05f2902 [bridge]
#14 [ffff90bcafac3720] __netif_receive_skb_core at ffffffff837eb50a
#15 [ffff90bcafac3798] __netif_receive_skb at ffffffff837ebd48
#16 [ffff90bcafac37b8] netif_receive_skb_internal at ffffffff837ebdd0
#17 [ffff90bcafac37e8] netif_receive_skb at ffffffff837ebe6c
#18 [ffff90bcafac3808] l2w_udp_rcv at ffffffffc05a12ca [l2w]
#19 [ffff90bcafac3828] __udp4_lib_rcv at ffffffff83863c9e
#20 [ffff90bcafac38c0] udp_rcv at ffffffff838648fa
#21 [ffff90bcafac38d0] ip_local_deliver_finish at ffffffff838316d9
#22 [ffff90bcafac38f8] ip_local_deliver at ffffffff838319c9
#23 [ffff90bcafac3950] ip_rcv_finish at ffffffff83831340
#24 [ffff90bcafac3978] ip_rcv at ffffffff83831cf9
#25 [ffff90bcafac39e0] __netif_receive_skb_core at ffffffff837eba39
#26 [ffff90bcafac3a58] __netif_receive_skb at ffffffff837ebd48
#27 [ffff90bcafac3a78] netif_receive_skb_internal at ffffffff837ebdd0
#28 [ffff90bcafac3aa8] netif_receive_skb at ffffffff837ebe6c
#29 [ffff90bcafac3ac8] br_netif_receive_skb at ffffffffc05f1f28 [bridge]
#30 [ffff90bcafac3ae0] br_pass_frame_up at ffffffffc05f2028 [bridge]
#31 [ffff90bcafac3b58] br_handle_frame_finish at ffffffffc05f2301 [bridge]
#32 [ffff90bcafac3bd8] br_handle_frame at ffffffffc05f2911 [bridge]
#33 [ffff90bcafac3c60] __netif_receive_skb_core at ffffffff837eb50a
#34 [ffff90bcafac3cd8] __netif_receive_skb at ffffffff837ebd48
#35 [ffff90bcafac3cf8] netif_receive_skb_internal at ffffffff837ebdd0
#36 [ffff90bcafac3d28] napi_gro_receive at ffffffff837ec9f8
#37 [ffff90bcafac3d50] e1000_receive_skb at ffffffffc0394f1f [e1000e]
#38 [ffff90bcafac3d88] e1000_clean_rx_irq at ffffffffc0396e5b [e1000e]
#39 [ffff90bcafac3e28] e1000e_poll at ffffffffc039eda2 [e1000e]
#40 [ffff90bcafac3e78] net_rx_action at ffffffff837ec3ef
#41 [ffff90bcafac3ef8] __do_softirq at ffffffff8329a945
#42 [ffff90bcafac3f68] call_softirq at ffffffff83922d2c
#43 [ffff90bcafac3f80] do_softirq at ffffffff8322d625
#44 [ffff90bcafac3fa0] irq_exit at ffffffff8329acc5
#45 [ffff90bcafac3fb8] do_IRQ at ffffffff83923fc6
--- <IRQ stack> ---
#46 [ffff90bca94e3db8] ret_from_intr at ffffffff83916362
    [exception RIP: cpuidle_enter_state+84]
    RIP: ffffffff83769364  RSP: ffff90bca94e3e60  RFLAGS: 00000202
    RAX: 000000701812f068  RBX: ffff90bca94e3e40  RCX: 0000000000000018
    RDX: 0000000225c17d03  RSI: ffff90bca94e3fd8  RDI: 000000701812f068
    RBP: ffff90bca94e3e88   R8: 0000000000000085   R9: ffff90bcafad7ad0
    R10: 7fffffffffffffff  R11: 000000aadcc5c800  R12: 0000000000000003
    R13: ffff90bcafad39e0  R14: ffffffff832be645  R15: ffff90bca94e3de0
    ORIG_RAX: ffffffffffffff19  CS: 0010  SS: 0018
#47 [ffff90bca94e3e90] cpuidle_idle_call at ffffffff837694be
#48 [ffff90bca94e3ed0] arch_cpu_idle at ffffffff832352fe
#49 [ffff90bca94e3ee0] cpu_startup_entry at ffffffff832f2b6a
#50 [ffff90bca94e3f28] start_secondary at ffffffff83255362
#51 [ffff90bca94e3f50] start_cpu at ffffffff832000d5
crash> mod -s l2w /root/l2w.ko
mod: l2w: last symbol: l2w_kill_thread is not _MODULE_END_l2w?
     MODULE       NAME                    SIZE  OBJECT FILE
ffffffffc05b73c0  l2w                   153093  /lib/modules/3.10.0.862.ac/kernel/net/bridge/l2w/l2w.ko 
crash> dis ffffffffc05a3b78 
0xffffffffc05a3b78 <l2w_vdev_hard_xmit_broad_pkt+248>:  mov    0xe8(%rax),%rdx
crash> sym 0xffffffffc05a3b78
ffffffffc05a3b78 (T) l2w_vdev_hard_xmit_broad_pkt+248 [l2w] /usr/src/kernels/3.10.0.862.ac/net/bridge/l2w/vdev.c: 216
crash> exit
[root@localhost ~]#crash>
```

3. 分析

在上面例子中进入crash后我分别执行了一下命令

+ bt
+ mod -s l2w /root/l2w.ko
+ dis  ffffffffc05a3b78
+ sym 0xffffffffc05a3b78
  
最终从上面信息中我得到了产生 **panic** 的具体代码是 **/usr/src/kernels/3.10.0.862.ac/net/bridge/l2w/vdev.c: 216**

4. 如何定位堆栈的位置？
   
通常使用 **bt** 命令查看内核crash前的堆栈信息中 **RIP** 寄存器的值就是指向最终造成panic 的地址，然后我们通过 **mod -s l2w /root/l2w.ko** 把我们的内核模块加载进入当前的crash内核中（注意：你要把模块事先放在指定位置，我这里是绝对路径 **/root/l2w.ko**）。 只有把模块加载后，才能进行反汇编。
使用 **dis** 命令对地址进行反汇编 ， 使用 **sym + 地址** 命令，物理地址映射出具体的符号（也就是我们代码的具体位置）

