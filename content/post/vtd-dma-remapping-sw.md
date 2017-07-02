+++
date = "2017-06-27T21:58:50+08:00"
title = "手撕intel-iommu之DMA remapping初始化"
draft = false

+++

我在DMA remapping相关的初始化函数中加入dump_stack()函数，以此来跟踪内核初始化过程中intel iommu之DMA remapping初始化流程。

```
start_kernel
	mem_init
		pci_iommu_alloc
			detect_intel_iommu
				dmar_table_detect
				dmar_walk_dmar_table
					for (iter = start; iter < end; iter = next) {
						dmar_validate_one_drhd
					}
				x86_init.iommu.iommu_init = intel_iommu_init
```
<font color=blue>`detect_intel_iommu`的作用是探测平台是否支持intel iommu功能</font>，其主要步骤如下：

+ 调用 dmar_table_detect 在ACPI表中查找是否有DMAR表
+ 调用 dmar_walk_dmar_table 对DMAR表的每一项执行 validate_drhd_cb 中指定的操作。此处会对每一表通过 dmar_validate_one_drhd 判断是否是合法
+ 设置iommu_init钩子为 intel_iommu_init

<br/>

```
kernel_init
	kernel_init_freeable
		do_one_initcall
			pci_iommu_init
				iommu_init钩子(intel_iommu_init)
					iommu_init_mempool
					dmar_table_init
						if (dmar_table_initialized == 0) {
							parse_dmar_table
							dmar_table_initialized = 1
						}
					dmar_dev_scope_init
					dmar_init_reserved_ranges
						reserve_iova(IOAPIC_RANGE_START, IOAPIC_RANGE_END)
						for_each_pci_dev(pdev) {
							reserve_iova(r->start, r->end)
						}
					init_no_remapping_devices
					init_dmars
					dma_ops = &intel_dma_ops;
					bus_set_iommu
					intel_iommu_enabled = 1
```
<font color=blue>`intel_iommu_init`对intel iommu进行全面的初始化</font>，其主要步骤如下：

+ 调用 iommu_init_mempool 初始化3个kmem_cache：iova_cache（"iommu_iova"）、iommu_domain_cache（"iommu_domain"）、iommu_devinfo_cache（"iommu_devinfo"）
+ 调用 dmar_table_init --> parse_dmar_table 对ACPI DMA Remapping相关的表（DRHD/RMRR/ATSR/RHSA/ANDD）进行解析<font color=red>具体每项如何解析未看...vt-d spec ch8</font>
+ 调用 dmar_dev_scope_init 将每个pci dev添加到所属的DMAR hardware unit中<font color=red>不明白</font>
+ 调用 dmar_init_reserved_ranges 将MSI地址区间及所有pci dev的MMIO区间加入reserved_iova_list，这些区间不能被remapping（前者vt-d ch3.13有说明；<font color=red>后者是为何？近期看到社区有推使能pcie p2p支持的patch([Enabling peer to peer device transactions for PCIe devices](https://lists.01.org/pipermail/linux-nvdimm/2017-January/008395.html))，难道是软件还不支持？</font>）
+ 调用 init_no_remapping_devices ：绝大多数gfx drivers不会调用standard PCI DMA APIs来分配DMA buffers，这与IOMMU有冲突。因此如果一个DMA remapping hardware unit中如果只有gdx devices，则根据cmdline配置来决定iommu是否需要将它们bypass掉。[原始patch](https://lkml.org/lkml/2007/4/24/226)
+ init_dmars 对 intel_iommu 做详细的初始化设置（具体见下文分析）
+ 调用 bus_set_iommu 设置pci bus的iommu_ops钩子为intel_iommu_ops，并注册了一个bus notifier —— iommu_bus_notifier

<br/>

```
init_dmars
	g_iommus = kcalloc(....)
	for_each_possible_cpu(cpu) {
		setup_timer(&dfd->timer, flush_unmaps_timeout, cpu)
	}
	for_each_active_iommu(iommu, drhd) {
		g_iommus[iommu->seq_id] = iommu
		intel_iommu_init_qi
			dmar_enable_qi
			iommu->flush.flush_context = qi_flush_context
			iommu->flush.flush_iotlb = qi_flush_iotlb
		iommu_init_domains
		iommu_alloc_root_entry
	}
	for_each_active_iommu(iommu, drhd) {
		iommu_set_root_entry
		iommu->flush.flush_context
		iommu->flush.flush_iotlb
	}
	if (iommu_identity_mapping) {
		si_domain_init
		iommu_prepare_static_identity_mapping
	}
	for_each_rmrr_units(rmrr) {
		for_each_active_dev_scope {
			iommu_prepare_rmrr_dev
		}
	}
	iommu_prepare_isa
```

+ 分配用于存储所有intel_iommu的数组空间
+ 初始化per-cpu的deferred_flush_data对象，它在 IOTLB invalid 操作时被使用
+ DMA Remapping转换过程中可能会有多种translation caches，在软件改变转换表时需要invalid相关old caches。vt-d中提供了两种invalid的方式：Register-based invalidation interface 和 Queued invalidation interface，如果需要支持irq remapping，则必须用后者，故我们分析后者（vt-d spec ch6.5.2）。<br/>对于平台上的每个active的iommu，通过 intel_iommu_init_qi 对其进行初始化设置：
	+ 分配相关数据结构，其中包括了一个作为Invalidation Queue的page
	+ 将DMAR_IQT_REG（Invalidation Queue Tail Register）设置为0
	+ 设置DMAR_IQA_REG（Invalidation Queue Address Register）：IQ的地址和大小
	+ 设置DMAR_GCMD_REG（Global Command Register）使能QI功能
	+ 等待DMAR_GSTS_REG（Global Status Register）的QIES置位，表示使能成功
	+ 设置flush.flush_context和flush.flush_iotlb两个钩子
+ 通过 iommu_init_domains 对intel_iommu数据结构中的domain_ids、domains域进行初始化
+ 通过 iommu_alloc_root_entry 分配一个用作iommu root entry的page，存储在iommu->root_entry中
+ 通过 iommu_set_root_entry 设置root table地址：
	+ 设置DMAR_RTADDR_REG（Root Table Address Register），设置root table基地址
	+ 写DMAR_GCMD_REG的SRTP位进行设置
+ 当cmdline中有“iommu=pt”（表示只对直通设备做DMA Remapping）时iommu_pass_through会设为1，init_dmars中编会设置iommu_identity_mapping。<br/>调用 si_domain_init 对si_domain（statically identity mapping domain）进行初始化<br/>调用 iommu_prepare_static_identity_mapping 将每个pci dev与si_domain关联起来
+ 对于RMRR中每一项锁包含的每一个dev，调用 iommu_prepare_rmrr_dev --> domain_prepare_identity_map 为其建立identity mapping
+ 调用 iommu_prepare_isa 为isa bridge建立identity mapping

注：最后两步骤中，在我的PC上并没有建立identity mapping，从结果可知：
```
[    0.995584] DMAR: Setting RMRR:
[    0.995585] DMAR: Ignoring identity map for HW passthrough device 0000:00:02.0 [0xc3800000 - 0xc7ffffff]
[    0.995585] DMAR: Ignoring identity map for HW passthrough device 0000:00:14.0 [0xc14c1000 - 0xc14e0fff]
[    0.995586] DMAR: Prepare 0-16MiB unity mapping for LPC
[    0.995586] DMAR: Ignoring identity map for HW passthrough device 0000:00:1f.0 [0x0 - 0xffffff]
[    0.995593] DMAR: Intel(R) Virtualization Technology for Directed I/O
```
domain_prepare_identity_map中有如下桥段：
```
   2652         if (domain == si_domain && hw_pass_through) {
   2653                 pr_warn("Ignoring identity map for HW passthrough device %s [0x%Lx - 0x%Lx]\n",
   2654                         dev_name(dev), start, end);
   2655                 return 0;
   2656         }
   2657
   2658         pr_info("Setting identity map for device %s [0x%Lx - 0x%Lx]\n",
   2659                 dev_name(dev), start, end);
   2660
```
并没有输出"Setting identity map for device..."，所以我的PC此时全走了if块中、即直接return了。<br/>**hw_pass_through**表示平台上的iommu硬件是否支持只对直通设备做remapping的功能。










