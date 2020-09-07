---
title: u-boot启动流程
type: uboot
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

## 启动流程

对于一块嵌入式板子，我们要寻找它的uboot启动流程，首先我们要清楚板子的CPU架构，这里可以查阅我的另一篇文章[uboot源码结构]()对于源码`arch`目录的介绍。 例如，我这里使用armv7架构，那么找启动流程入口的话，我么直接看 `arch/arm/cpu/armv7/start.S`文件就可以了。

### 第一个文件：arch/arm/cpu/armv7/start.S

在走读 `start.S`过程中，我们重点关注 `b` 和 `bl` 跳转指令，就能摸清楚启动流程了

```
#include <asm-offsets.h>
#include <config.h>
#include <asm/system.h>
#include <linux/linkage.h>

/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 * Do important init only if we don't start from memory!
 * Setup memory and board specific bits prior to relocation.
 * Relocate armboot to ram. Setup stack.
 *
 *************************************************************************/

	.globl	reset
	.globl	save_boot_params_ret

@uboot的启动就成就是从这个reset开始的（当然，reset之前还有一部分，但我们不需要关心了，因为这部分厂商都是做好的）
reset:
	/* Allow the board to save important registers */

  @  b 指令跳转，和 bl指令的区别是：b指令跳过去就不回来了，bl指令是先保存现场，跳转后再回来
  @ 这里使用b指令跳转，难道真的不回来了吗？
  @ 我们往下找到 标号 save_boot_params
	b	save_boot_params

  @ 上面跳转后，又回来了
save_boot_params_ret:
	/*
	 * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
	 * except if in HYP mode already
   * 关闭中断
	 */
	mrs	r0, cpsr
	and	r1, r0, #0x1f		@ mask mode bits
	teq	r1, #0x1a		@ test for HYP mode
	bicne	r0, r0, #0x1f		@ clear all mode bits
	orrne	r0, r0, #0x13		@ set SVC mode
	orr	r0, r0, #0xc0		@ disable FIQ and IRQ
	msr	cpsr,r0

/* Setup CP15 barrier */
#if defined (CONFIG_ARCH_IPQ807x) || defined (CONFIG_ARCH_IPQ806x) || defined (CONFIG_ARCH_IPQ6018)
	mrc p15, 0, r0, c1, c0, 0 @Read SCTLR to r0
	orr r0, r0, #0x20 @set the cp15 barrier enable bit
	mcr p15, 0, r0, c1, c0, 0 @write back to SCTLR
#endif

/*
 * Setup vector:
 * (OMAP4 spl TEXT_BASE is not 32 byte aligned.
 * Continue to use ROM code vector only in OMAP4 spl)
 */
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
	/* Set V=0 in CP15 SCTLR register - for VBAR to point to vector */
	mrc	p15, 0, r0, c1, c0, 0	@ Read CP15 SCTLR Register
	bic	r0, #CR_V		@ V = 0
	mcr	p15, 0, r0, c1, c0, 0	@ Write CP15 SCTLR Register

	/* Set vector address in CP15 VBAR register */
	ldr	r0, =_start
	mcr	p15, 0, r0, c12, c0, 0	@Set VBAR
#endif

	/* the mask ROM code should have PLL and others stable */

  /* 这里有两个bl 调用，既然使用了宏开关，那么可以理解为，可有可无，那么暂且不关心，因为这部分不影响整体启动流程*/
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_cp15
	bl	cpu_init_crit
#endif

  /* bl _main ，按照惯例，main函数其实就是一个进程的入口了，那么这个 标号在哪里呢？  
  * 找不到的话，我们直接在`arch/arm` 路径下搜索 `_main`就可以了
  * 最终我们发现 在 arch/arm/lib/crt0.S 有 _main的定义，接下来启动流程就切换到 crt0.S了
  */
	bl	_main
ENTRY(c_runtime_cpu_setup)
/*
 * If I-cache is enabled invalidate it
 */
#ifndef CONFIG_SYS_ICACHE_OFF
	mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache
	mcr     p15, 0, r0, c7, c10, 4	@ DSB
	mcr     p15, 0, r0, c7, c5, 4	@ ISB
#endif

	bx	lr

ENDPROC(c_runtime_cpu_setup)

/*************************************************************************
 *
 * void save_boot_params(u32 r0, u32 r1, u32 r2, u32 r3)
 *	__attribute__((weak));
 *
 * Stack pointer is not yet initialized at this moment
 * Don't save anything to stack even if compiled with -O0
 *
 *************************************************************************/
ENTRY(save_boot_params)
  @ 跳到这个标号了，然后又跳到 save_boot_params_ret，其实就是又回到reset了
  @ 为什么这样？  根据前面解释，主要是让你来存储一些数据的，因为如果使用bl跳转，也可以保存现场，但是保存什么数据不是你自己定的。 等于说，这里留了一个扩展性的接口
	b	save_boot_params_ret		@ back to my caller
ENDPROC(save_boot_params)
```

### 第二个文件：arch/arm/lib/crt0.S

对于这个文件，我们也是重点关注 `b` 和 `bl` 指令

```
ENTRY(_main)

/*
 * Set up initial C runtime environment and call board_init_f(0).
 */

#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
	ldr	sp, =(CONFIG_SPL_STACK)
#else
	ldr	sp, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
#if defined(CONFIG_CPU_V7M)	/* v7M forbids using SP as BIC destination */
	mov	r3, sp
	bic	r3, r3, #7
	mov	sp, r3
#else
	bic	sp, sp, #7	/* 8-byte alignment for ABI compliance */
#endif
	mov	r0, sp

  /*第一次跳转：这个符号我们在 .S 文件中没有找到，那会是在哪里呢？
  * 我们打开uboot代码找找看有没有这个函数，因为函数编译完成之后，函数名就是一个标号
  * 果然找到 `common/init/board_init.c`
  */
	bl	board_init_f_mem
	mov	sp, r0

#if defined(CONFIG_IPQ_NO_RELOC)
	ldr	r0, =__bss_start	/* this is auto-relocated! */

#ifdef CONFIG_USE_ARCH_MEMSET
	ldr	r3, =__bss_end		/* this is auto-relocated! */
	mov	r1, #0x00000000		/* prepare zero to clear BSS */

	subs	r2, r3, r0		/* r2 = memset len */
	bl	memset
#else
	ldr	r1, =__bss_end		/* this is auto-relocated! */
	mov	r2, #0x00000000		/* prepare zero to clear BSS */

clbss_l:cmp	r0, r1			/* while not at end of BSS */
#if defined(CONFIG_CPU_V7M)
	itt	lo
#endif
	strlo	r2, [r0]		/* clear 32-bit BSS word */
	addlo	r0, r0, #4		/* move to next */
	blo	clbss_l
#endif
#endif /* CONFIG_ARCH_IPQ807x clear bss */
	mov	r0, #0

  /*这个跳转是重点，common/board_f.c*/
	bl	board_init_f

#if ! defined(CONFIG_SPL_BUILD)

/*
 * Set up intermediate environment (new sp and gd) and call
 * relocate_code(addr_moni). Trick here is that we'll return
 * 'here' but relocated.
 */

	ldr	sp, [r9, #GD_START_ADDR_SP]	/* sp = gd->start_addr_sp */
#if defined(CONFIG_CPU_V7M)	/* v7M forbids using SP as BIC destination */
	mov	r3, sp
	bic	r3, r3, #7
	mov	sp, r3
#else
	bic	sp, sp, #7	/* 8-byte alignment for ABI compliance */
#endif
	ldr	r9, [r9, #GD_BD]		/* r9 = gd->bd */
	sub	r9, r9, #GENERATED_GBL_DATA_SIZE /* new GD is below bd */

	adr	lr, here
	ldr	r0, [r9, #GD_RELOC_OFF]		/* r0 = gd->reloc_off */
	add	lr, lr, r0
#if defined(CONFIG_CPU_V7M)
	orr	lr, #1				/* As required by Thumb-only */
#endif
	ldr	r0, [r9, #GD_RELOCADDR]		/* r0 = gd->relocaddr */
	b	relocate_code
here:
/*
 * now relocate vectors
 */

	bl	relocate_vectors

/* Set up final (full) environment */

	bl	c_runtime_cpu_setup	/* we still call old routine here */
#endif
#if !defined(CONFIG_SPL_BUILD) || defined(CONFIG_SPL_FRAMEWORK)
# ifdef CONFIG_SPL_BUILD
	/* Use a DRAM stack for the rest of SPL, if requested */
	bl	spl_relocate_stack_gd
	cmp	r0, #0
	movne	sp, r0
# endif
#if !defined(CONFIG_IPQ_NO_RELOC)
	ldr	r0, =__bss_start	/* this is auto-relocated! */

#ifdef CONFIG_USE_ARCH_MEMSET
	ldr	r3, =__bss_end		/* this is auto-relocated! */
	mov	r1, #0x00000000		/* prepare zero to clear BSS */

	subs	r2, r3, r0		/* r2 = memset len */
	bl	memset
#else
	ldr	r1, =__bss_end		/* this is auto-relocated! */
	mov	r2, #0x00000000		/* prepare zero to clear BSS */

clbss_l:cmp	r0, r1			/* while not at end of BSS */
#if defined(CONFIG_CPU_V7M)
	itt	lo
#endif
	strlo	r2, [r0]		/* clear 32-bit BSS word */
	addlo	r0, r0, #4		/* move to next */
	blo	clbss_l
#endif
#endif /* CONFIG_ARCH_IPQ807x  bss is already cleared */

#if ! defined(CONFIG_SPL_BUILD)
  /*这里跳转到led 配置，我们可以利用这一点在uboot阶段点亮led等
  * 这两个函数在代码中只是打桩，我们可以自己实现覆盖它们
  * common/board_f.c
  */
	bl coloured_LED_init
	bl red_led_on
#endif
	/* call board_init_r(gd_t *id, ulong dest_addr) */
	mov     r0, r9                  /* gd_t */
	ldr	r1, [r9, #GD_RELOCADDR]	/* dest_addr */
	/* call board_init_r */

  /*最后调用 board_init_r  从此uboot一去不复返
  * common/board_r.c
  * 在board_init_r中会遍历数组 init_sequence_r 中所有的函数指针，最后调用
  * run_main_loop 要么进入交互模式，或者启动内核
  */
	ldr	pc, =board_init_r	/* this is auto-relocated! */

	/* we should not return here. */
#endif

ENDPROC(_main)
```