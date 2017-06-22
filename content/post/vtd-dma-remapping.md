+++
date = "2017-06-22T20:18:38+08:00"
title = "手撕intel-iommu之DMA remapping（硬件篇）"
draft = false

+++
由于KVM虚拟机的设备直通方案目前基本都采用vfio方式，而vfio下层依赖于平台的iommu功能，因此我决定研究下intel iommu。计划根据Intel vt-d spec和Linux内核相关源码（主要在`drviers/iommu/`下面）进行学习。

## DMA Remapping
### Types of DMA requests
Remapping硬件将来自于 集成在root-complex中 或 挂载在PCIE bus上的 设备的memory request分成两类：

+ **Requests-without-PASID**：这是来自于endpoint devices的普通memory requests。它们一般指明了访问类型(read/write/atomics)、DMA目的地的地址与大小、源设备标识。

+ **Requests-with-PASID**：这是来自于支持virtual memory capabilities的endpoint devices的memory requests。它们除了指明上述普通信息外，还带有附加信息：目标PASID、扩展属性等...

（注：PASID全名Process Address Space ID，来源于PCIE中的概念）

### Domains and Address Translation
Domain是一个抽象的定义，表示平台上一个隔离的、从host physical memory中分配的子集。对于虚拟化来说，软件可以将每个虚拟机看作是一个单独的domain。

Remapping硬件的作用是：在硬件进行进一步处理（例如：address decoding, snooping of processor caches, and/or forwarding to the memory controllers）之前，将memory request中的地址转换成host physical address。

### Mapping Devices to Domains
#### Source Identifier
**source-id**：用于标识I/O transaction的发起者。
对于PCIE设备，其source-id位于PCI-Express transaction layer header中，由Bus/Device/Function组成。

![](https://nimisolo.github.io/iommu-source-id.jpg)

#### Root-Entry & Extended-Root-Entry
Root-table是devices mapping的最顶层结构。它的起始地址由**Root Table Address Register**指定，共计4-KB大小，由256个root-entries组成。每个root-entry对应一个bus number，所以source-id中的bus number在mapping过程中会被当做root-table中的index来使用。
我们首先来看看Root Table Address Register的详细描述：

![](https://nimisolo.github.io/iommu-rtar.jpg)

![](https://nimisolo.github.io/iommu-rtar-2.jpg)






