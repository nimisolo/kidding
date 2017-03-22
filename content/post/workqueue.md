+++
date = "2017-03-22T20:38:22+08:00"
title = "Linux Concurrency Managed Workqueue分析"
draft = false

+++

### 概述

### 核心数据结构

### 框架分析

#### workqueue子系统初始化

##### workqueue_init_early

此函数对workqueue子系统做早期初始化。它会建立某些数据结构及创建系统的workquues，其他模块的早期初始化代码在这之后便可以queue/cancel work items了，但是这些work items只有在相关worker kthread建立（会在`workqueue_init`中做）之后才能够得到运行。

### <font color=red>关键问题</font>

#### worker的亲和性能改变吗

#### worker何时唤醒/休眠

#### rescuer作用及运行时间

#### timers的作用

#### worker何种情况下会扩建

#### worker何时销毁

#### work太多了能做负载均衡吗