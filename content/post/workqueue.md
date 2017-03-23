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
因为普通worker pool是per-cpu的，其cpumask也就只含有相应cpu，在创建work线程时会用它来指定线程的cpu亲和性，所以<font color=red>对于普通work poll来说，其work线程是只能在相应的cpu上运行，不能migrate</font>

##### workqueue_init

此函数是workqueue子系统初始化的第二步（也是最后一步）。
它首先会为每个cpu的每个普通worker pool设置好其所在的node，存储在`pool->node`中；
然后通过`create_worker`为每个普通worker pool创建一个worker（包括一个工作线程），worker创建初始化完成后会通过`worker_enter_idle`进入WORKER_IDLE状态；
除了普通worker pool之外，系统中此时可能已有ubound worker pool（`workqueue_init_early`之后其他子系统早期代码中可能会创建），此时也需要通过`create_worker`为它们创建worker。

#### worker创建




### <font color=red>关键问题</font>

#### worker的亲和性能改变吗

#### worker何时唤醒/休眠

#### rescuer作用及运行时间

#### timers的作用

#### worker何种情况下会扩建

#### worker何时销毁

#### work太多了能做负载均衡吗