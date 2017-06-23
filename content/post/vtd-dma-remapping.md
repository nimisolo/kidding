+++
date = "2017-06-22T20:18:38+08:00"
title = "手撕intel-iommu之DMA remapping（硬件篇）"
draft = false

+++
由于KVM虚拟机的设备直通方案目前基本都采用vfio方式，而vfio下层依赖于平台的iommu功能，因此我决定研究下intel iommu。计划根据Intel vt-d spec和Linux内核相关源码（主要在`drviers/iommu/`下面）进行学习。

## <font color=red>DMA Remapping</font>
### <font color=red>Types of DMA requests</font>
Remapping硬件将来自于 集成在root-complex中 或 挂载在PCIE bus上的 设备的memory request分成两类：

+ **Requests-without-PASID**：这是来自于endpoint devices的普通memory requests。它们一般指明了访问类型(read/write/atomics)、DMA目的地的地址与大小、源设备标识。

+ **Requests-with-PASID**：这是来自于支持virtual memory capabilities的endpoint devices的memory requests。它们除了指明上述普通信息外，还带有附加信息：目标PASID、扩展属性等...

（注：PASID全名Process Address Space ID，来源于PCIE中的概念）

### <font color=red>Domains and Address Translation</font>
Domain是一个抽象的定义，表示平台上一个隔离的、从host physical memory中分配的子集。对于虚拟化来说，软件可以将每个虚拟机看作是一个单独的domain。

Remapping硬件的作用是：在硬件进行进一步处理（例如：address decoding, snooping of processor caches, and/or forwarding to the memory controllers）之前，将memory request中的地址转换成host physical address。

### <font color=red>Mapping Devices to Domains</font>
#### <font color=red>Source Identifier</font>
**source-id**：用于标识I/O transaction的发起者。
对于PCIE设备，其source-id位于PCI-Express transaction layer header中，由Bus/Device/Function组成。

![](https://nimisolo.github.io/iommu-source-id.JPG)

#### <font color=red>Root-Entry & Extended-Root-Entry</font>
Root-table是devices mapping的最顶层结构。它的起始地址由**Root Table Address Register**指定，共计4-KB大小，由256个root-entries组成（可以算出：每个root-entry有16个字节空间）。每个root-entry对应一个bus number，所以source-id中的bus number在mapping过程中会被当做root-table中的index来使用。

##### <font color=red>Root Table Address Register</font>

我们首先来看看Root Table Address Register的详细描述：

![](https://nimisolo.github.io/iommu-rtar.JPG)

![](https://nimisolo.github.io/iommu-rtar-2.JPG)

可以看到：有两种形式的table，根据RTT bit进行控制。下面就依次来看这两种table。

##### <font color=red>Root-table entry</font>

![](https://nimisolo.github.io/iommu-rte.JPG)

![](https://nimisolo.github.io/iommu-rte-2.JPG)

##### <font color=red>Extended Root-table entry</font>
在支持Extended-Context-Support (ECS=1 in Extended Capability Register)的硬件上，如果RTADDR_REG的RTT=1，则它指向的是一个extended root-table。

![](https://nimisolo.github.io/iommu-extended-rte.JPG)

![](https://nimisolo.github.io/iommu-extended-rte-2.JPG)

综上，又会涉及到两种context entry，下面依次来看。

##### <font color=red>regular Context-Entry</font>
一个context-entry用于将一个bus上的一个特定I/O device映射到其所属的domain中，也就是指向该domain的地址翻译结构。

source-id中的低8位（device#和function#）用来作为设备在context-table中的index使用。

![](https://nimisolo.github.io/iommu-map.JPG)

<font color=blue>Context-entries只能支持requests-without-PASID</font>，它详细结构如下：

![](https://nimisolo.github.io/iommu-ce.JPG)

##### <font color=red>Extended-Context-Entry</font>