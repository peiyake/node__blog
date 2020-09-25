---
title: linux中断子系统---中断处理流程
type: kernel
description: linux中断处理流程分为上半部和下半部，上半部程序退出后会唤醒下半部程序，然后下半部会在合适的时机执行
date: 2020-09-16 13:57:54
---

关于中断子系统，网上[蜗蜗科技]()有很详细深入的介绍。本人在拜读蜗蜗的文章后，自己也做一些总结，加深印象。

[请点这里，找蜗蜗科技,中断子系统系列文章](http://www.wowotech.net/sort/irq_subsystem)


## linux中断子系统示意流程图

![中断子系统](/images/softirq.png)

## 上半部处理流程

**do_IRQ**

`do_IRQ`函数是依赖体系结构的，在linux源码中不同硬件平台的实现不一样，但整体都差不多，都会调用`irq_enter`、`generic_handle_irq`、`irq_exit`，下面举个例子看一下：

```
unsigned int do_IRQ(int irq, struct uml_pt_regs *regs)
{
	struct pt_regs *old_regs = set_irq_regs((struct pt_regs *)regs);
	irq_enter();
	generic_handle_irq(irq);
	irq_exit();
	set_irq_regs(old_regs);
	return 1;
}
```

* set_irq_regs 保存中断发生时，一些必要的寄存器信息，用于中断返回时恢复现场
* irq_enter 进入中断上下文
* generic_handle_irq调用注册的中断的上半部处理程序handle
* irq_exit退出中断上下文
* set_irq_regs恢复现场，继续原来的工作

`irq_enter`进入中断上下文之后，做了什么？

进入irq_enter先判断如果是调度空闲上下文 并且 不在中断上下文（软/硬中断），那么会先屏蔽软中断进行 `tick_irq_enter`，之后开启软中断。最后调用 `_irq_enter`，记录中断进入的时间用于统计，将**preempt_count**值的硬中断标记位 +1，(这样就保证，硬中断执行期间不会被打断)

```
/*
 * Enter an interrupt context.
 */
void irq_enter(void)
{
	rcu_irq_enter();
	if (is_idle_task(current) && !in_interrupt()) {
		/*
		 * Prevent raise_softirq from needlessly waking up ksoftirqd
		 * here, as softirq will be serviced on return from interrupt.
		 */
		local_bh_disable();
		tick_irq_enter();
		_local_bh_enable();
	}

	__irq_enter();
}
/*
 * It is safe to do non-atomic ops on ->hardirq_context,
 * because NMI handlers may not preempt and the ops are
 * always balanced, so the interrupted value of ->hardirq_context
 * will always be restored.
 */
#define __irq_enter()					\
	do {						\
		account_irq_enter_time(current);	\
		preempt_count_add(HARDIRQ_OFFSET);	\
		trace_hardirq_enter();			\
	} while (0)
```

然后调用generic_handle_irq调用注册的中断的上半部处理程序handle

```
/**
 * generic_handle_irq - Invoke the handler for a particular irq
 * @irq:	The irq number to handle
 *
 */
int generic_handle_irq(unsigned int irq)
{
	struct irq_desc *desc = irq_to_desc(irq);

	if (!desc)
		return -EINVAL;
	generic_handle_irq_desc(desc);
	return 0;
}
/*
 * Architectures call this to let the generic IRQ layer
 * handle an interrupt.
 */
static inline void generic_handle_irq_desc(struct irq_desc *desc)
{
	desc->handle_irq(desc);
}
```

最后调用 `irq_exit`，记录硬中断处理结束时间用来统计；将**preempt_count** -1；再判断如果当前不在中断上下文 并且有下半部中断在等待处理，那么调用`invoke_softirq`唤醒软中断。最后调用`tick_irq_exit`和`rcu_irq_exit`退出。 

```
/*
 * Exit an interrupt context. Process softirqs if needed and possible:
 */
void irq_exit(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
	local_irq_disable();
#else
	WARN_ON_ONCE(!irqs_disabled());
#endif

	account_irq_exit_time(current);
	preempt_count_sub(HARDIRQ_OFFSET);
	if (!in_interrupt() && local_softirq_pending())
		invoke_softirq();

	tick_irq_exit();
	rcu_irq_exit();
	trace_hardirq_exit(); /* must be last! */
}
```

至此，一次硬中断的上半部处理过程结束。


## 下半部处理流程

在讲解下半部流程之前，先说明一下下半部程序是如何组织的。

事实上，在内核初始化时，会为每个CPU创建一个线程`ksoftirqd/%u`，这里%u就是对应CPU core id，线程的信息用结构体 `struct smp_hotplug_thread`描述，如下：

```
static struct smp_hotplug_thread softirq_threads = {
	.store			= &ksoftirqd,
	.thread_should_run	= ksoftirqd_should_run,
	.thread_fn		= run_ksoftirqd,
	.thread_comm		= "ksoftirqd/%u",
};
```

可以看到，线程的入口统一为 `run_ksoftirqd`，该函数会去检测当前是否有软中断pedding，如果有的话调用 `__do_softirq`来处理下半部注册函数。

>注意：在 `kernel/softirq.c`中通过内核宏 `early_initcall`来提早注册内核线程


也就是说有两种情况会触发中断下半部的执行：

1. 在上半部流程中，最后`irq_exit`在退出前，检测是如果不处于中断上下文，并且有软中断pendding的话，直接唤醒软中断。
2. 内核线程`ksoftirqd/%u`来轮询检测是否存在软中断需要处理

下面介绍，软中断处理过程，

首先是，内核线程入口 `run_ksoftirqd`

```
static void run_ksoftirqd(unsigned int cpu)
{
	local_irq_disable();
	if (local_softirq_pending()) {
		/*
		 * We can safely run softirq on inline stack, as we are not deep
		 * in the task stack here.
		 */
		__do_softirq();
		local_irq_enable();
		cond_resched_rcu_qs();
		return;
	}
	local_irq_enable();
}
```

我们看到，线程运行后首先关闭了全局中断（此时硬中断无法触发），如果有pending，调用 `__do_softirq`。

看到这里有个疑问：关闭全局中断，难道软中断运行中，硬中断真的无法触发了？带着以为继续看 `__do_softirq`

```
asmlinkage __visible void __do_softirq(void)
{
//////////////////////////////////第一部分//////////////////////////////////////////////////////////////
	unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
	unsigned long old_flags = current->flags;
	int max_restart = MAX_SOFTIRQ_RESTART;
	struct softirq_action *h;
	bool in_hardirq;
	__u32 pending;
	int softirq_bit;

	/*
	 * Mask out PF_MEMALLOC s current task context is borrowed for the
	 * softirq. A softirq handled such as network RX might set PF_MEMALLOC
	 * again if the socket is related to swap
	 */
	current->flags &= ~PF_MEMALLOC;

	pending = local_softirq_pending();
	account_irq_enter_time(current);
	__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
///////////////////////////////////第二部分//////////////////////////////////////////////////////////////////

	in_hardirq = lockdep_softirq_start();

restart:
	/* Reset the pending bitmask before enabling irqs */
	set_softirq_pending(0);

	local_irq_enable();
////////////////////////////////////第三部分/////////////////////////////////////////////////////////////////
	h = softirq_vec;

	while ((softirq_bit = ffs(pending))) {
		unsigned int vec_nr;
		int prev_count;

		h += softirq_bit - 1;

		vec_nr = h - softirq_vec;
		prev_count = preempt_count();

		kstat_incr_softirqs_this_cpu(vec_nr);

		trace_softirq_entry(vec_nr);
		STOPWATCH_START(softirq[vec_nr]);
		h->action(h);
		STOPWATCH_STOP(softirq[vec_nr]);
		trace_softirq_exit(vec_nr);
		if (unlikely(prev_count != preempt_count())) {
			pr_err("huh, entered softirq %u %s %p with preempt_count %08x, exited with %08x?\n",
			       vec_nr, softirq_to_name[vec_nr], h->action,
			       prev_count, preempt_count());
			preempt_count_set(prev_count);
		}
		h++;
		pending >>= softirq_bit;
	}

	rcu_bh_qs();
	local_irq_disable();

//////////////////////////////////////第四部分//////////////////////////////////////////////////////////
	pending = local_softirq_pending();
	if (pending) {
		if (time_before(jiffies, end) && !need_resched() &&
		    --max_restart)
			goto restart;

		wakeup_softirqd();
	}

	lockdep_softirq_end(in_hardirq);
	account_irq_exit_time(current);
///////////////////////////////////////第五部分/////////////////////////////////////////////////////////////
	__local_bh_enable(SOFTIRQ_OFFSET);
	WARN_ON_ONCE(in_interrupt());
	tsk_restore_flags(current, old_flags, PF_MEMALLOC);
}
```

我们用 **//**将`__do_softirq`分成5个部分，一个一个来看：

**第一部分**，因为在进入`__do_softirq`之前调用了`local_irq_disable`，所以在第一部分，全局中断关闭，此时无法再产生新的硬中断。而第一步分的工作只是做了很少量的工作。重点是最后调用`__local_bh_disable_ip`，用来屏蔽软中断（这里说明，软中断是串行执行的，软中断执行期间不能被其它软中断打断，但是可以被硬中断打断。注意：这里只是只单个cpu上）。屏蔽软中断后进入第二部分。
**第二部分**，重点是调用 `local_irq_enable`开启全局中断，开启后如果再有硬中断产生，那么此时就会被打断

**第三部分**，进入第三部分，此时全局中断是打开的。第三部分重点是 while循环中调用 `h->action(h);` 这里其实就是调用内核注册的每个软中断类型的action函数，在action函数中再执行我们注册的软中断处理函数。关于软中的部分，另外还有文章讲解

**第四部分**，在第三部分退出前调用`local_irq_disable`，再次屏蔽了全局中断，因为第四部分的内容不能被打断，第四部分主要作用是，下面三个条件同时满足再restart 软中断的处理
  
  1. softirq的处理时间没有超过2个ms
  2. 上次的softirq中没有设定TIF_NEED_RESCHED，也就是说没有有高优先级任务需要调度
  3. loop的次数小于 10次

**第五部分**，重点是`__local_bh_enable`，开启软中断，此时允许其它软中断处理。


