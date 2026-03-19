---
title: Linux物理内存管理
date: 2025-07-19 00:04:28
tags: 内存
categories: 操作系统
---

## 一、在 CPU 角度看物理内存模型

内核是以页为基本单位堆物理内存进行管理，将物理内存划分为一个个以一页 4k 为大小的内存块。每个内存块使用 struct_page 进行管理，struct_page 中封装了每页内存块的状态信息。为了快速索引到对应的物理内存页，内存为每个页 struct_page 定义了一个索引编号：PFN(Page Frame Number), 还定义了两个宏来完成 PNF 索引与物理内存页结构体 struct page 之间相互转换，分别是 page_to_pfn, pfn_to_page。内核如何管理这些页面 struct_page 结构体可以称之为物理内存模型，对于不同的物理内存模型，page_to_pfn, pfn_to_page 的计算逻辑是不同的。

### FLATMEM 平坦内存模型

将一片物理内存划分为固定大小且连续的内存块，每个内存块对应的 strcut_page 也是连续的。所以用一个数组来组织这些连续的物理内存页 struct_page 结构体，把数组的下标当做 PFN 索引。

![](./JuTIbps74oBWx6xOX8tcO0hWnEd.png)

内存中使用一个全局数组 mem_map 来组织所有物理内存页，在该内存模型下的 page_to_pfn，pfn_to_page 的计算就是对数组 mem_map 进行偏移操作。

![](./NZVtbOpxro0fpxxlDDPcEeGRntc.png)

ARCH_PFN_OFFSET 就是 PFN 的起始偏移量。

Linux 早期使用的就是这种内存模型，因为在 Linux 发展的早期所需要管理的物理内存通常不大（比如几十 MB）那时的 Linux 使用平坦内存模型 FLATMEM 来管理物理内存就足够高效了。内核中的默认配置是使用 FLATMEM 平坦内存模型。

### DISCONTIGMEM 非连续内存模型

为了组织和管理不连续的物理内存，内核引入 DISCONTIGMEM 非连续内存模型。在 DISCONTIGMEM 非连续内存模型中，内核将物理内存划分为一个个 node 节点，每个 node 节点管理一块连续的物理内存，既然 node 节点的管理的物理内存是连续的就可以使用 FLATMEME 平坦内存模型来组织管理物理内存页，每个 node 节点中包含一个 struct page* node_mem_map 数组来管理连续的物理内存页。

![](./Znh7b0G2goZyhyx645rceI6Mnag.png)

对于 page_to_pfn 和 pfn_to_page 的计算逻辑就比 FLATMEME 内存模型多了一步定位 page 所在的 node 的操作。实际过程中通过 arch_pfn_to_nid 可以根据物理页的 PFN 定位到物理页 page 所在的 node, 和通过 page_to_nid 将物理页 page 转化为 page 所在的 node。定位到 node 结构后剩下的逻辑与 FLATMEME 内存模型就一样了。

![](./C1qlbGCbjox5wtxYCKhconienDd.png)

### SPARSEMEM 稀疏内存模型

随着内存技术的发展，内核可以支持物理内存的热插拔了，这样一来物理内存的不连续就变为常态了，在上小节介绍的 DISCONTIGMEM 内存模型中，其实每个 node 中的物理内存也不一定都是连续的，所以需要对连续物理内存颗粒度有更细的需求。SPARSEMEME 稀疏内存模型的核心思想就是对粒度更小的连续的内存块进行精细的管理，用于管理的内存块单元称为 section。 在物理页大小为 4k 的情况下，section 大小为 128M。内核中使用 mem_section 结构体来表示这个 section。

![](./YTltbERu1oVCkjxQXBmcnRk7nJg.png)

由于 section 内部管理的是连续的物理内存，所以可以通过数组来管理这些内存。每个 mem_section 结构体中都会有一个 section_mem_map 指针用于指向 section 中管理的连续内存的 page 数组。

SPARSEME 内存模型将全部 mem_section 会存放在一个全局数组 mem_section 中，并且每个 mem_section 都可以在系统运行时改变上线下线状态，以便支持内存热插拔的功能。

![](./LtB9b5HFBoey7vxccS1cAeTvntf.png)

![](./QbItbaubnoe25Yx4UnWcScMQn5U.png)

在 SPARSEMEM 稀疏内存模型下 page_to_pfn 与 pfn_to_page 的计算逻辑:

page_to_pfn :根据 struct page 结构定位到 mem_section  数组中具体的 section 结构。然后在通过 section_mem_map 定位到具体的 PFN。

pfn_to_page: 首先需要通过 __pfn_to_section 根据 PFN 定位到 mem_section 数组中具体的 section 结构。然后在通过 PFN 在 section_mem_map 数组中定位到具体的物理页 Page 。
![](./Pi04bBgDMoZGhZxKyUScLm7PnDc.png)


#### 物理内存热插拔

前面提到随着内存技术的发展，物理内存的热插拔 hotplug 在内核中得到了支持，由于物理内存可以动态的从主板中插入以及拔出，所以导致了物理内存的不连续已经成为常态。因此内核引入了 SPARSEMEM 稀疏内存模型以便应对这种情况，提供对更小粒度的连续物理内存的灵活管理能力。

从整体上来讲，内存热插拔分为两个阶段：

物理热插拔阶段：这个阶段主要是从物理上将内存硬件插入（hot-add),拔出（hot-remove）主板的过程，其中涉及到硬件和内核的支持。

逻辑热插拔阶段： 这个阶段主要是内核中内存管理子系统来负责，涉及到的主要工作为：如何动态上线启用（online)刚刚插入的内存，如何动态下线刚刚拔出的内存。

物理内存拔出比插入需要多处理很多的事情，因为此时即将要被拔出的物理内存中可能已经为进程分配了物理页，如何妥善安置这些已经被分配的物理页是一个棘手的问题。

前边我们介绍 SPARSEMEM 内存模型的时候提到，个 mem_section 都可以在系统运行时改变 offline ，online 状态，以便支持内存的热插拔（hotplug）功能。当 mem_section offline 时， 内核会把这部分内存隔离开，使得这部分内存不可以再次被使用，然后把这部分内存中已经分配的内存页面迁移到其他 mem_section 的内存上。

![](./AQLMbAOjIoOw5qxaj3mcelDrnqb.png)

但会出现一个问题，就是并非所有的物理页都可以迁移。因为迁移意味着物理地址会发生变化，而内存热插拔对于进程来说是透明的，进程无法感知到内存的状态，所以对于迁移后的物理内存映射的虚拟内存地址是不能变化的。

这一点在进程的用户空间是没问题的，可以通过更新页表中虚拟地址和物理地址的映射关系，可以保证虚拟地址不会改变。

但是在内核的虚拟地址空间中有一段直接映射区，在这段虚拟地址区域中虚拟地址与物理地址是直接映射的关系，虚拟地址通过减去一个偏移量得到物理地址。这就会造成如果物理地址变化其映射的虚拟地址也会发生变化，对应的物理页就没法进行迁移。

内核通过给内存分类来就解决这个问题。 内核将内存按照是否可以迁移划分为 不可迁移页，可回收页，可迁移页。然后去设置将不可迁移的内存不可拔出，将那些可能被拔出的页分配为可以迁移的内存页。这样可以拔出的内存都是可以迁移的，从而使内存可以安全的拔出。

## 二、从 CPU 角度看物理内存架构

### 一致性内存访问 UMA 架构

![](./Mg5BbHZRioRQevxYZ50cgf7fnne.png)

在 UMA 架构下，多个 CPU 位于总线的一侧，所有内存组成的大片内存位于总线的另一侧，所有 CPU 访问内存都要过总线，而且距离都是一样的，所以所有 CPU 访问内存的速度是一样的。

但是随着多核技术的发展，服务器上的 CPU 数量会越来越多，而 UMA 架构下所有的 CPU 都是通过总线来访问内存 ，这样总线就会很快成为性能瓶颈，主要体现在两个方面：

1.总线带宽的压力会越来越大， 随着 CPU 个数的增多导致每个 CPU 的带宽会减少。

2.总线的长度也会增加，进而增加访问的延迟。

为了解决以上问题，提高 CPU 访问内存的性能和扩展性，引入了一种新的架构： 非一致性内存访问 NUMA。

### 非一致性内存访问 NUMA 架构

在 NUMA 架构下，内存就不是一整片的了，而是被划分成了一个一个的内存节点 ，每个 CPU 都有属于自己的本地内存节点，CPU 和自己的本地内存一起构成 NUMA 节点。CPU 访问自己本地内存是不需要经过总线的，因此访问速度是最快的。在本地内存不够用的时候，CPU 也可以跨节点去访问其他内存，这样 CPU 访问内存的速度就要慢一些，访问不同内存速度不一致所以叫做非一致性内存访问架构。

![](./Rggub2o0PoTCVZxLIRUcFMrQnsc.png)

CPU 与 CPU 之间通过 QPI 点对点完成互联，在 CPU 的本地内存不足的情况下，CPU 需要通过 QPI 访问远程 NUMA 节点上的内存控制器从而在远程内存节点上分配内存，这就导致了远程访问比本地访问多了额外的延迟开销。

#### NUMA 的内存分配策略

NUMA 的内存分配策略是指在 NUMA 架构下下 CPU 如何请求内存分配的相关策略。

![](./OJiLb7icVoUkSxxKId0cjwFqn1f.png)

我们可以在应用程序中通过 libnuma 共享库中的 API 调用 set_mempolicy 接口设置进程的内存分配策略。

![](./CUQIbRB6zo7nifxVqgyc2HzunGd.png)

mode : NUMA 内存分配策略； nodemask ：指定的 NUMA 节点 ID； maxnode ：指定最大 NUMA 节点 Id，用于遍历远程节点，实现跨 NUMA 节点分配内存。

#### NUMA 的使用介绍

在我们理解了物理内存的 NUMA 架构，以及在 NUMA 架构下的内存分配策略之后，本小节我来为大家介绍下如何正确的利用 NUMA 提升我们应用程序的性能。

##### 查看 NUMA 相关信息

`numactl -H` 命令可以查看服务器的 NUMA 配置

![](./SZuCbUbQeoHAu0xqi5Tc2KsdnKg.png)

上图中的服务器配置共包含 4 个 NUMA 节点（0 - 3），每个 NUMA 节点中包含 16 个 CPU 核心，本地内存大小约为 64G。`node distances:` 这一栏，node distances 给出了不同 NUMA 节点之间的访问距离，角线上的值均为本地节点的访问距离 10 。

`numactl -s` 来查看 NUMA 的内存分配策略设置：

![](./ZVeCb8uiPoUGkuxcDUqcxFAonFh.png)

`numastat` 还可以查看各个 NUMA 节点的内存访问命中率：

![](./NSz3bj74soTaUKxhPCPcaErqnbf.png)

- numa_hit ：内存分配在该节点中成功的次数。
- numa_miss : 内存分配在该节点中失败的次数。
- numa_foreign：表示其他 NUMA 节点本地内存分配失败，跨节点（numa_miss）来到本节点分配内存的次数。
- interleave_hit : 在 MPOL_INTERLEAVE 策略下，在本地节点分配内存的次数。
- local_node：进程在本地节点分配内存成功的次数。
- other_node：运行在本节点的进程跨节点在其他节点上分配内存的次数。

## 三、内核如何管理 NUMA 节点

在 NUMA 架构下，只有 DISCONTIGMEM 非连续内存模型和 SPARSEMEM 稀疏内存模型是可用的。而 UMA 架构下，前面介绍的三种内存模型均可以配置使用。

无论是 NUMA 架构还是 UMA 架构在内核中都是使用相同数据结构来组织管理的，在内核的内存管理模块中会把 UMA 架构当做只有一个 NUMA 节点的伪 NUMA 架构。这样一来这两种架构模式就在内核中被统一管理起来。

### 内核如何组织 NUMA 节点

内核使用 pglist_data 来描述 NUMA 节点。在内核 2.4 版本之间，使用 pgdat_list 单链表将 pglist_data 串联起来。在内核 2.4 版本之后，使用的使用全局数组 node_data 来管理这些 NUMA 节点。

![](./D4RObsLlmoYOVkxqhgfcmMGQnde.png)

pglist_data 结构

![](./R5LVbNYAkodeLgxaLNBcfbttnxd.png)

### NUMA 节点物理内存区域划分

![](./YDZObsBmfom9BVxmGWgc4uNJnpd.png)

1. ZONE_DMA：用于那些无法对全部物理内存进行寻址的硬件设备，进行 DMA 时的内存分配。
2. ZONE_DMA32：与 ZONE_DMA 区域类似，该区域内的物理页面可用于执行 DMA 操作，不同之处在于该区域是提供给 32 位设备（只能寻址 4G 物理内存）执行 DMA 操作时使用的。该区域只在 64 位系统中起作用，因为只有在 64 位系统中才会专门为 32 位设备提供专门的 DMA 区域。
3. ZONE_NORMAL：这个区域的物理页都可以以直接映射到内核中的虚拟内存，由于是线性映射，内核可以直接进行访问。
4. ZONE_HIGHMEM：这个区域包含的物理页就是我们说的高端内存，内核不能直接访问这些物理页，这些物理页需要动态映射进内核虚拟内存空间中。
5. ZONE_DEVICE 是为支持热插拔设备而分配的非易失性内存，也可用于内核崩溃时保存相关的调试信息。
6. ZONE_MOVABLE 是内核定义的一个虚拟内存区域，该区域中的物理页可以来自于上边介绍的几种真实的物理区域。。该区域中的页全部都是可以迁移的，主要是为了防止内存碎片和支持内存的热插拔。

### NUMA 节点的内存规整和回收

内存是使用的过程中会经历回收不经常使用的内存页面和迁移页面来腾出连续的物理内存空间。

内核会给每个 NUMA 节点分配一个 kswapd 进程来回收不经常使用的页面 和 分配一个 kcompactd 进程用于内存规整避免内存碎片。

![](./NwsRbYgYzoC2DUxOMHzcFzFTnkg.png)

### NUMA 节点状态 node_states

如果系统中的 NUMA 节点多于一个，内核会维护一个位图 node_states，用于维护各个 NUMA 节点的状态信息。

![](./XBs4bzczwo4AkLx8kqwcthTrnze.png)

1. N_POSSIBLE 表示 NUMA 节点在某个时刻可以变为 online 状态。
2. N_ONLINE 表示 NUMA 节点当前的状态为 online 状态。
3. N_NORMAL_MEMORY 表示节点没有高端内存，只有 ZONE_NORMAL 内存区域。
4. N_HIGH_MEMORY 表示节点有 ZONE_NORMAL 内存区域或者有 ZONE_HIGHMEM 内存区域。
5. N_MEMORY 表示节点有 ZONE_NORMAL，ZONE_HIGHMEM，ZONE_MOVABLE 内存区域。
6. N_CPU 表示节点包含一个或多个 CPU。

## 四、内核如何管理 NUMA 节点中的物理内存区域

NUMA 节点的描述符 struct pglist_data 过 struct zone 类型的数组 node_zones 将 NUMA 节点中划分的物理内存区域连接起来。

![](./HuEubepWuoFWqPxM0nvc61Qzn6d.png)

### 物理内存区域中的预留内存

除了前边介绍的关于物理内存区域的这些基本信息之外，每个物理内存区域 struct zone 还为操作系统预留了一部分内存，这部分预留的物理内存用于内核的一些核心操作，这些操作无论如何是不允许内存分配失败的。

### 物理内存区域中的水位线

内存资源是系统中最宝贵的系统资源，是有限的。当内存资源紧张的时候，系统的应对方法无非就是三种：

1. 产生 OOM，内核直接将系统中占用大量内存的进程，将 OOM 优先级最高的进程干掉，释放出这个进程占用的内存供其他更需要的进程分配使用。
2. 内存回收，将不经常使用到的内存回收，腾挪出来的内存供更需要的进程分配使用。
3. 内存规整，将可迁移的物理页面进行迁移规整，消除内存碎片。从而获得更大的一片连续物理内存空间供进程分配。

内核会为每个 NUMA 节点中的每个物理内存区域定制三条用于指示内存容量的水位线，分别是：WMARK_MIN（页最小阈值）， WMARK_LOW （页低阈值），WMARK_HIGH（页高阈值）。

### 水位线的计算

通常情况下 WMARK_LOW 的值是 WMARK_MIN 的 1.25 倍，WMARK_HIGH 的值是 WMARK_LOW 的 1.5 倍。而 WMARK_MIN 的数值就是由这个内核参数 min_free_kbytes 来决定的。

min_free_kbytes 的计算逻辑定义在 `init_per_zone_wmark_min` 方法中

![](./GolUbz9iOo14XSxvj0pcrnImnAf.png)

### 物理内存区域中的冷热页

所谓的热页就是已经加载进 CPU 高速缓存中的物理内存页，所谓的冷页就是还未加载进 CPU 高速缓存中的物理内存页，冷页是热页的后备选项。

在 2.6.25 版本之前的内核源码中，物理内存区域 struct zone 包含了一个 struct per_cpu_pageset 类型的数组 pageset。其中内核关于冷热页的管理全部封装在 struct per_cpu_pageset 结构中。

## 五、内核如何描述物理内存页

首先关于物理页面的大小，Linux 规定必须是 2 的整数次幂，因为 2 的整数次幂可以将一些数学运算转换为移位操作，比如乘除运算可以通过移位操作来实现，这样效率更高。在内存紧张的时候，内核会将不经常使用到的物理页面进行换入换出等操作，内存与磁盘之间传输小块数据时会更加的高效，所以综上所述内核会采用 4KB 作为默认物理内存页大小。

### 匿名页的反向映射

通常所说的内存映射是正向映射，即从虚拟内存到物理内存的映射。

而反向映射则是从物理内存到虚拟内存的映射，用于当某个物理内存页需要进行回收或迁移时，此时需要去找到这个物理页被映射到了哪些进程的虚拟地址空间中，并断开它们之间的映射。

### 内存页回收相关属性

物理内存页在内核中分为匿名页和文件页，其中 active 链表用来存放访问非常频繁的内存页（热页），inactive 链表用来存放访问不怎么频繁的内存页（冷页），当内存紧张的时候，内核就会优先将 inactive 链表中的内存页置换出去。

### 物理内存页属性和状态的标志位 flag

flags 字段的高 8 位用来表示 struct page 的定位信息，剩余低位表示特定的标志位。

![](./KDk2bdUfToTaXuxqnLockIbYnef.png)

### 复合页 compound_page 相关属性

Linux 管理内存的最小单位是 page，每个 page 描述 4K 大小的物理内存，但在一些对于内存敏感的使用场景中，用户往往期望使用一些巨型大页。

因为这些巨型页要比普通的 4K 内存页要大很多，所以遇到缺页中断的情况就会相对减少，由于减少了缺页中断所以性能会更高。

虽然巨型页（compound_page）是由多个物理上连续的普通 page 组成的，但是在内核的视角里它还是被当做一个特殊内存页来看待。

![](./MqBYbTPDqoDlVNxktxfchM5vndd.png)

### Slab 对象池相关属性

内核中对内存页的分配使用有两种方式，，一种是一页一页的分配使用，这种以页为单位的分配方式内核会向相应内存区域 zone 里的伙伴系统申请以及释放。

另一种方式就是只分配小块的内存，不需要一下分配一页的内存，通常就是几十个字节大小。为了满足类似这种小内存分配的需要，Linux 内核使用 slab allocator 分配器来分配，slab 就好比一个对象池，内核中的数据结构对象都对应于一个 slab 对象池，用于分配这些固定类型对象所需要的内存。

它的基本原理是从伙伴系统中申请一整页内存，然后划分成多个大小相等的小块内存被 slab 所管理。
