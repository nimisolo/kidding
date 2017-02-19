+++
date = "2017-02-19T08:42:58+08:00"
title = "Linux内核idle进程分析"
draft = false
tags = ["kernel", "KVM", "idle", "MWAIT"]

+++

### 写作背景
在当前KVM实现中，当vcpu中执行HLT指令时会发生VMexit，KVM模块中会进入`kvm_vcpu_block`函数，从前此函数会快速将vcpu线程schedule out，但在配置了halt_poll_ns的情况下，当条件满足时会适当的busy loop一段时间，代码如下：
```
void kvm_vcpu_block(struct kvm_vcpu *vcpu)
{
	...
		do {
			/*
			 * This sets KVM_REQ_UNHALT if an interrupt
			 * arrives.
			 */
			if (kvm_vcpu_check_block(vcpu) < 0) {
				++vcpu->stat.halt_successful_poll;
				if (!vcpu_valid_wakeup(vcpu))
					++vcpu->stat.halt_poll_invalid;
				goto out;
			}
			cur = ktime_get();
		} while (single_task_running() && ktime_before(cur, stop));	
	...
}
```
当vcpu此时不再是halt状态 或 当前pcpu上不止一个可运行task 或 busy loop时间到期 这三者任一满足时将退出busy loop，在busy loop过程中CPU使用率肯定会冲高到100%。可是我们不想将这段时间被TOP工具即pmcint工具（利用PMU）统计到，前者可以通过修改内核cputime.c中的几个统计代码来实现，但是后者的话稍微有点麻烦。pmcint工具利用了PMU硬件打点的原理，我们没法修改统计，最后想到的办法是改造此busy loop，在其中进入C1 state，这样的话PMU计数器会暂停，从而使得pmcint暂停打点，这样既确保了vcpu性能，又无法使用户看到很高的占用率。

细想一下，此处busy loop的好处是啥？无非就是在条件满足的情况下不要将vcpu线程切换出去，否则来回切换势必使得vcpu线程被唤醒不够及时，导致虚拟机性能受影响。这样来说的话，我们在busy loop中调用HLT/MWAIT也能起到一样的效果：

+ 如果使用MWAIT，我们可以用MONITOR监控让busy loop结束的变量，这样当需要结束时MWAIT会被breaken，从而与此前流程一样。

+ 如果使用HLT，则我们需要根据stop设置一个backend hrtimer，保证我们肯定能够从HLT中出来。（但是，这个方式肯定不如上述代码的实现，因为HLT的breaken事件中不包括内存监控，它没法及时监控到nr_running的变化，这可能会导致调度不及时）


### CPU Cstates
Cstates是ACPI规范中引入的，具体的可以看acpi spec，下面内容摘自[Haswell芯光大道之六：C-States十种状态解析](http://www.expreview.com/25426.html)
![](intel-power-state.jpg)

### MONITOR/MWAIT指令

#### 会不会有安全问题
<font color=red>**问题：**</font>虚拟化下，如果guest kernel中执行了MONITOR gva指令，此时会不会影响到host中正处于mwait状态的cpu？例如：无论是某个vcpu STORE了该gva、或者某个pcpu STORE了某hva（而此hva的数值与该gva相等），都将引起mwait BROKEN?

<font color=blue>**解决：**</font>

+ intel SDM vol3 26.3.3 / 27.5.6 说，VMentry/VMexit时会将可能有影响的所有“address-range monitoring”清除。（题外话：什么叫“可能有影响”？**[a]**我觉得都是“可能有影响的”，因为该address-range是个virtual address，对于64位host+64位guest的线性地址空间大小是一样的，硬件上无法区分，所以应该是都会清除。**[b]**但是这样的话又不对了，假如pcpu监控了hva1，然后它进入了guest，这时需要将hva1清除，当VMexit后并没有提到恢复对hva1的监控，那么此pcpu上后续STORE hva1不就不能被监控了吗？**[c]**除非有种可能，address-range由至少二元组确定，即virtual address + cpu mode，但是这样的话SDM何不直接描述为“guest或non-root的address-range清除”即可？<font color=red>哎...暂时不懂</font>）

+ intel SDM vol3 25.1.3 说，如果VM-execution control的“MONITOR exiting”为1，则MONITOR指令会导致VMexit。

在KVM中，在`setup_vmcs_config`中“MONITOR exiting”是会强制为1的，所以vcpu只要执行该指令就会引起VMexit，退出处理`handle_monitor`中直接跳过该指令、把它当做NOP来处理。


### Linux idle进程

#### 执行框架分析

#### intel_idle分析
