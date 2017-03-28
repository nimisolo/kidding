+++
date = "2017-03-22T20:38:22+08:00"
title = "Linux Concurrency Managed Workqueue分析"
draft = false
tags = ["kernel", "workqueue", "kthread"]

+++

### 概述

### 核心数据结构

### 框架分析

#### workqueue子系统初始化

##### workqueue_init_early

此函数对workqueue子系统做早期初始化。它会建立某些数据结构及创建系统的workquues，其他模块的早期初始化代码在这之后便可以queue/cancel work items了，但是这些work items只有在相关worker kthread建立（会在`workqueue_init`中做）之后才能够得到运行。

此函数中最核心一步是<font color=red>为每个cpu初始化了两个普通worker pools（一个nice=0的、一个nice=-20的）</font>，这些worker pool都存在per-cpu变量中：
```
/* the per-cpu worker pools */
static DEFINE_PER_CPU_SHARED_ALIGNED(struct worker_pool [NR_STD_WORKER_POOLS], cpu_worker_pools);
```

<font color=red>实际上，无论是普通worker-pool还是unbound/ordered worker-pool，都有低优先级和高优先级两种。</font>

因为普通worker pool是per-cpu的，其cpumask也就只含有相应cpu，在创建work线程时会用它来指定线程的cpu亲和性，所以<font color=red>对于普通work poll来说，其work线程是只能在相应的cpu上运行，不能migrate</font>

##### workqueue_init

此函数是workqueue子系统初始化的第二步（也是最后一步）。

它首先会为每个cpu的每个普通worker pool设置好其所在的node，存储在`pool->node`中；

然后通过`create_worker`为每个普通worker pool创建一个worker（包括一个工作线程），worker创建初始化完成后会通过`worker_enter_idle`进入WORKER_IDLE状态；

除了普通worker pool之外，系统中此时可能已有ubound worker pool（`workqueue_init_early`之后其他子系统早期代码中可能会创建），此时也需要通过`create_worker`为它们创建worker。

#### worker创建

##### create_worker

此函数用于为@pool创建一个worker。主要步骤如下：

1）通过`alloc_worker`分配一个worker对象;

2）调用`kthread_create_on_node`函数创建该worker的工作线程，工作线程的执行体是**`worker_thread`**;

<font color=red>**注意：**</font>`kthread_create_on_node`会通过`kthreadd_task`线程在指定node上（具体哪个cpu上不可测）创建一个内核线程。

3）设置工作线程的优先级（`set_user_nice`）;

4）将工作线程绑定至指定的cpumask（`kthread_bind_mask`）;

<font color=red>**注意：**</font>`kthread_bind_mask`中会无条件将线程的allowed_cpumask设置为新的cpumask，但即使线程当前所处的cpu不在新的cpumask中也不会做迁移操作。

5）通过`worker_attach_to_pool`函数将此worker与@pool关联起来，加入到@pool的workers链表中;

<font color=red>**注意：**</font>`worker_attach_to_pool`中调用了`set_cpus_allowed_ptr`函数，但是对于worker kthread来说调用它是没有实际意义的，因为它会检查allowed_cpumask是否与new_cpumask相等，如果相等的话什么也不干，
```
int set_cpus_allowed_ptr(struct task_struct *p, const struct cpumask *new_mask)
{
	return __set_cpus_allowed_ptr(p, new_mask, false);
}

static int __set_cpus_allowed_ptr(struct task_struct *p,
				  const struct cpumask *new_mask, bool check)
{
	......
	if (cpumask_equal(&p->cpus_allowed, new_mask))
	goto out;
	......
}
```
但这里为什么需要调用呢？`worker_attach_to_pool`在rescuer_thread也会调用。一个workqueue可以对应多个worker pool，但是只有一个rescuer线程，当其某个worker pool需要“急救”时，rescuer_thread通过·将自己“派到一线”去救援。

对于worker thread来说，只有等到被调度运行的时候才有机会被迁移到合适的cpu上运行（try_to_wake_up-->select_task_rq）。





### <font color=red>关键问题</font>

#### worker的亲和性能改变吗

worker线程在创建中会通过`kthread_bind_mask`设置它的allowed_cpumask，并且会设置PF_NO_SETAFFINITY标记。

改变亲和性的内核函数是`__set_cpus_allowed_ptr`,
```
static int __set_cpus_allowed_ptr(struct task_struct *p,
				  const struct cpumask *new_mask, bool check)
{
	......
	if (check && (p->flags & PF_NO_SETAFFINITY)) {
		ret = -EINVAL;
		goto out;
	}
	......
}
```
如果`check`为真，则不能设置带`PF_NO_SETAFFINITY`标记的线程的亲和性。

<font color=red>特殊的，</font>rescuer线程也是worker，它在运行过程中会调用`worker_attach_to_pool`根据具体情况而调整自己的亲和性，而：
```
static void worker_attach_to_pool(struct worker *worker,
				   struct worker_pool *pool)
{
	......
	set_cpus_allowed_ptr(worker->task, pool->attrs->cpumask);
	......
}

int set_cpus_allowed_ptr(struct task_struct *p, const struct cpumask *new_mask)
{
	return __set_cpus_allowed_ptr(p, new_mask, false);
}
```
可以看到传入的check=false，即即使带PF_NO_SETAFFINITY标记也要修改，所以，<font color=red>rescuer线程在运行过程中会动态调整自己的亲和性，除此之外内核不会主动调整其他worker线程的亲和性</font>。

用户态程序（例如taskset）可通过`sched_setaffinity`来改变线程的亲和性，但是它会首先判断线程是否有PF_NO_SETAFFINITY标记，有的话则不能修改亲和性，
```
long sched_setaffinity(pid_t pid, const struct cpumask *in_mask)
{
	......
	if (p->flags & PF_NO_SETAFFINITY) {
		retval = -EINVAL;
		goto out_put_task;
	}
	......
}
```
所以，<font color=red>用户态程序不能修改worker线程的亲和性。</font>


#### worker何时唤醒/休眠

#### rescuer作用及运行时间

#### timers的作用

#### worker何种情况下会扩建

#### worker何时销毁

#### work太多了能做负载均衡吗