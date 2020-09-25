---
title: linux中断子系统---preempt_count
type: kernel
description: preempt_count是一个32bit的整数，内核把这32bit分开来作为中断的各种标记，用来判断当前所处的终端上下文，以及是否有中断程序再执行
date: 2020-09-16 16:57:54
---

## preempt_count 定义

`linux/preemtp.h`

如下图所示：

![preempt含义](/images/preempt.png)

## 各字段含义

**HARDIRQ**: 根据前文 [linux中断子系统---中断处理流程](https://peiyake.com/2020/09/16/kernel/linux中断子系统---中断处理流程/)的讲解，但内核进入硬中断时调用 `irq_entry`，会调用 `preempt_count_add(HARDIRQ_OFFSET)`，此时 `preempt_count`的 **HARDIRQ**段会 + 1，也就是说如果不为0，说明此时正在有硬中断正在执行。通过宏 `in_irq()`来检测。

**PREEMPT**: 用来记录内核调度是否被禁止。在中断上下文中，调度是关闭的，不会发生进程的切换，这属于一种隐式的禁止调度，而在代码中，也可以使用preempt_disable()来显示地关闭调度，关闭次数由第0到7个bits来记录。每使用一次preempt_disable()，值就会加1，使用preempt_enable()则会让值减1。占8个bits，因此一共可以表示最多256层调度关闭的嵌套。

**SOFTIRQT**: 它记录了进入softirq的嵌套次数，如果softirq count的值为正数，说明现在正处于softirq上下文中。每使用一次local_bh_disable()，值就会加1，使用local_bh_enable()则会值减1。 使用宏 `in_softirq()`来检测

**NMI**:

## 中断上下文

中断上下文分为，硬中断上下文、软中断上下文，如果以 `preempt_count`来判断，那么只要是 **NMI** + **HARDIRQ** + **SOFTIRQT** + **PREEMPT**的值不为0，都可称为**当前是在中断上下文**

**判断宏**：`in_interrupt()`

### 硬中断上下文

**判断宏**：`in_irq()`，preempt_count的**HARDIRQ**域不为0

### 软中断上下文

**判断宏**：`in_softirq()`

这里引用 [蜗蜗科技]()上的一段话

>softirq context并没有那么的直接，一般人会认为当sofirq handler正在执行的时候就是softirq context。这样说当然没有错，sofirq handler正在执行的时候，会增加softirq count，当然是softirq context。不过，在其他context的情况下，例如进程上下文中，有有可能因为同步的要求而调用local_bh_disable，这时候，通过local_bh_disable/enable保护起来的代码也是执行在softirq context中。当然，这时候其实并没有正在执行softirq handler。如果你确实想知道当前是否正在执行softirq handler，in_serving_softirq可以完成这个使命，这是通过操作preempt_count的bit 8来完成的。


## 部分代码

```
/*
 * Are we doing bottom half or hardware interrupt processing?
 * Are we in a softirq context? Interrupt context?
 * in_softirq - Are we currently processing softirq or have bh disabled?
 * in_serving_softirq - Are we currently processing softirq?
 */
#define in_irq()		(hardirq_count())
#define in_softirq()		(softirq_count())
#define in_interrupt()		(irq_count())
#define in_serving_softirq()	(softirq_count() & SOFTIRQ_OFFSET)
```

