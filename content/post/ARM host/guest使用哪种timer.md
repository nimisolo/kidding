+++
date = "2017-02-05T16:45:06+08:00"
title = "ARM host/guest使用哪种timer"
draft = false
tags = ["kernel", "虚拟化", "ARM"]

+++

ARM Generic Timer支持好几种timer，此前听说ARM linux host使用pyhsical timer，而guest使用virtual timer。<font color=red>一直不清楚ARM linux中是怎么探测和设置它的，今晚看了下有所眉目了。</font>

首先来看个全局变量：
```
static enum ppi_nr arch_timer_uses_ppi = VIRT_PPI;
```
`arch_timer_uses_ppi`表示使用的timer类型，默认是virtual timer。

在`arch_timer_init`中根据情况会对此作出调整：
```
	if (is_hyp_mode_available() || !arch_timer_ppi[VIRT_PPI]) {
		bool has_ppi;

		if (is_kernel_in_hyp_mode()) {
			arch_timer_uses_ppi = HYP_PPI;
			has_ppi = !!arch_timer_ppi[HYP_PPI];
```
可以看到，对于host来说，会调整成physical timer。

在`__arch_timer_setup`中会对所使用的clocksource做设置：
```
		switch (arch_timer_uses_ppi) {
		case VIRT_PPI:
			clk->set_state_shutdown = arch_timer_shutdown_virt;
			clk->set_state_oneshot_stopped = arch_timer_shutdown_virt;
			clk->set_next_event = arch_timer_set_next_event_virt;
			break;
		case PHYS_SECURE_PPI:
		case PHYS_NONSECURE_PPI:
		case HYP_PPI:
			clk->set_state_shutdown = arch_timer_shutdown_phys;
			clk->set_state_oneshot_stopped = arch_timer_shutdown_phys;
			clk->set_next_event = arch_timer_set_next_event_phys;
			break;
		default:
			BUG();
		}
```
针对不同的timer（physical或virtual），设置了不同的钩子处理函数。这样一来，以后对clock进行操作的时候就各走各的路了，从而区分开来了。
