+++
date = "2017-02-05T15:54:39+08:00"
title = "关于periodic-mode timer的问题"
draft = false
tags = ["kernel", "timer"]

+++

今天和晓建讨论一个方案时，他说“即使定时器是periodic mode，当加入hrtimer时也有可能会去设置硬件计数器（tmict/tsc_deadline）的值”。
咦？！和我以前的认识不一样啊，根据以前的掌握：periodic mode情况下新加hrtimer时不会设置硬件计数器了，及时设置计数器是oneshot mode的做法啊！
于是重新翻了下相关代码，证明**<font color=red>晓建理解错了、我的观点是正确的</font>**。请看下面分析。

我们以新加一个定时器的操作入手：`hrtimer_start --> hrtimer_start_range_ns
`，在后者中有这么一段：
```
	leftmost = enqueue_hrtimer(timer, new_base);
	if (!leftmost)
		goto unlock;

	if (!hrtimer_is_hres_active(timer)) {
		/*
		 * Kick to reschedule the next tick to handle the new timer
		 * on dynticks target.
		 */
		if (new_base->cpu_base->nohz_active)
			wake_up_nohz_cpu(new_base->cpu_base->cpu);
	} else {
		hrtimer_reprogram(timer, new_base);
	}
```
1. 通过`enqueue_hrtimer`将（新添加的）hrtimer插入到当前的timerqueue（其实是一个rb-tree）中。如果该hrtimer是“将会最早到期”的，则它会放在rb-tree的最左端，即leftrmost为1；否则leftmost为0。
2. 如果不是最左端，则直接返回。这是因为：此函数是perdioc mode和oneshot mode公用的，如果此定时器将最早到期，oneshot mode会根据起到期时间重设硬件计数器。
3. 如果没有启动高精度定时器（`hrtimer_is_hres_active`返回false），则不会重设硬件计数器；否则（oneshot mode）就通过`hrtimer_reprogram`重设。

现在需要弄清楚的就是：`hrtimer_is_hres_active`什么情况下返回true。
```
static inline int hrtimer_is_hres_active(struct hrtimer *timer)
{
	return timer->base->cpu_base->hres_active;
}
```
实际上就是看`hres_active`是否为1。因此下面搜索下，看什么情况下会设置它为1。
```
static void hrtimer_switch_to_hres(void)
{
	struct hrtimer_cpu_base *base = this_cpu_ptr(&hrtimer_bases);

	if (tick_init_highres()) {
		printk(KERN_WARNING "Could not switch to high resolution "
				    "mode on CPU %d\n", base->cpu);
		return;
	}
	base->hres_active = 1;
	......
}
```
当`tick_init_highres`返回false时，才会设置`hres_active`为1。看名字，该函数的作用是初始化高精度定时器（即oneshot mode），深入该函数，可以发现：如果成功切换成oneshot，则会返回0。
好，到这里其实我们已经弄清楚了：
<font color=red>
+ 高精度highres其实是指onshot-mode。
+ periodic mode下，即使新加一个leftmost的hrtimer也不会重设硬件计数器。只有onshot mode下才会。
</font>

**更进一步**，当hrtimer到期之后，是如何被处理的？
**对于periodic-mode模式：**`tick_handle_periodic --> tick_periodic --> update_process_times --> run_local_timers --> hrtimer_run_queues --> __hrtimer_run_queues`
**对于oneshot-mode模式：**`hrtimer_interrupt --> __hrtimer_run_queues`

处理超时的hrtimer是最终都是在**`__hrtimer_run_queues`**函数中完成，<font color=red>它会（根据当前时间）遍历所有已经超时的hrtimer，然后处理它们</font>。

**再进一步**，为啥periodic-mode精度比oneshot-mode低低低？？
+ 因为periodic-mode下时钟中断触发时间是周期性的，也就是说在一个周期内到期的hrtimer只有在周期结束（时钟终端触发）时才能获得处理，显然实时性较差；并且如果一个周期内很多hrtimer到期，则依次处理它们，那么排在靠后的hrtimer实际上延迟就更大了。
+ oneshot-mode能够精确的根据leftmost hrtimer的到期时间来精确设置计数器，这样更加实时；并且根据此原理，一般多个hrtimer同时到期的情况应该比periodic-mode的少，则“依次处理”不会太长，这一点也比periodic-mode情况要好。
