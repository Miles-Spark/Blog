---
title: Unity垃圾回收机制--BoehmGC机制之内存回收
date: 2025-07-13 18:43:12
tags: Unity
categories: 游戏引擎
---



前文 Unity 垃圾回收机制--BoehmGC 机制之内存分配介绍了 GC 机制中内存分配的相关过程，本文主要分析 BoehmGC 机制中内存回收部分的具体实现。

## 回收流程

首先 BeohmGC 提供垃圾回收入口 API GC_gcollect, 然后通过 GC_gcollect 调用 GC_try_to_collect_general 调用 GC_try_to_collect_inner;

![](./Q2dLbTxxioN2jpxHUtucdhxknLf.png)

在 GC_try_to_collect_inner 中才开始真正的 GC 操作。

![](./VPOJbBZ1HoxRUPx28oqcqK7inXd.png)

GC 回收主要分为两个阶段

第一阶段：标记（mask)阶段，在这个阶段会先重置内存节点的标记位，然后从根对象出发遍历扫描内存空间，标记托管堆上被引用的内存节点。

第二个阶段： 回收（reclaim)阶段，在这个阶段会遍历托管堆上所有的内存节点，将未被标记的内存节点回收。

除此之外，GC 还提供了事件机制通过外部 GC 的状态

![](./GpxKbPDkIooMl3xKI2Oc4Pyznec.png)

外界可以在事件中添加回调，在 GC 的各个阶段会通过 GC_on_collection_event 进行通知。

## Mark 阶段

mark 阶段的目的是将托管堆上的内存标记为引用或非引用，便于后序回收阶段处理。

### 标记为重置

在 header 信息中记录了内存块中所有内存对象节点是否被引用

![](./WWPibAz9io9pKUxyd6jcdJ0Hnob.png)

其中，hb_n_marks 记录了内存块中被标记的个数，也就是被引用的内存对象的个数。hb_marks 维护了所有内存对象的引用标记为，用 0,1 表示引用或非引用。

![](./BSM5bBUCioo5BCxMKHCc2Fe0n13.png)

首先通过 GC_clear_marks 方法重置所有标记位，在方法中调用 GC_apply_to_all_blocks 遍历得到所有托管堆中的 hblk 内存块地址，通过内存块地址找到与之对应的 header 信息。 然后调用 clear_marks_for_block 重置内存块的标记，在 clear_marks_for_bloack 中首先通过 HDR 查找 header 信息，然后将 header 信息传入 GC_clear_hdr_marks 中将 header 中内存块节点标记和标记个数重置为 0。

### 扫描流程

重置完标记信息后进入扫描标记阶段，主要由 GC_stopped_mark 实现。

![](./EmRfbvpr7oRUBfx9G1jcSHajnzb.png)

首先进入准备工作，暂停线程；然后从根对象开始扫描，判断根对象内存地址指向的空间是否位于一个有效的 hblk 内存块中，将这些有效的内存空间来放入一个临时数组中；接下来遍历这个临时数组，将对应的内存空间进行标记； 最后结束标记流程，恢复线程。

#### GCThread

为了精准采集托管堆中内存的瞬时状态，就需要将当前线程挂起。GC 算法中为每个线程封装了一个 GC_THREAD 结构，结构中存储了线程相关的信息，包含了线程的 id, 线程的状态，栈等信息。

![](./V5M2bvzGdolAptxMQSxcqtuKn9b.png)

在创建线程的时候会获取栈底指针； 在线程处理暂停信号时，会将栈顶指针保存到 GC_THREAD 中； stop_info 中保存了线程暂停相关的信息 ；

#### suspendAll

在 GC_stopped_mask 中通过 STOP_WORLD()暂停线程，具体实现是通过定义宏 STOP_WORLD, 调用 GC_stop_world，再调用 GC_supend_all 来暂停除主线程外的所有线程

![](./Lh4xbS4vSoVdOrxgjdpcSnRZndf.png)

因为还需要线程来运行 GC 算法，所以会保留主线程，同时也会过滤结束和阻塞的线程，然后发布线程中断信号。

#### suspend_handler

接收到中断信号后，触发 GC_suspend_handler 回调函数，内部再调用 GC_suspend_hander_innder 方法。

![](./HLGqbM7AtoaaE3x55Ekc8qg1n9f.png)

#### 内存扫描

暂停完线程后才开始从根节点扫描，执行 GC_mark_some 方法从根节点开始扫描。

![](./XEIMbaMwIoqHMExoBuYcIIdXnSc.png)

GC_mark_state 表示当前扫描的状态，主要分为四个状态：

MS_INVALID ：表示内存空间未标记，此状态将会切换到　MS_PUSH_UNCOLLECTABLE 状态。

MS_PUSH_UNCOLLECTABLE：　开始遍历内存根节点，从根节点起始地址开始，逐渐偏移字节，判断根节点内存中指向的地址是否是有效的托管堆中的地址，如果是则托管堆中的内存为引用。

MS_ROOTS_PUSHED：遍历之间被引用的托管堆中内存，按照偏移地址的方式，判断指向的地址是否也是托管堆中的地址。

MS_NONE：　表示扫描全部结束，全部标记结束。

#### 根节点扫描

在 MS_PUSH_UNCOLLECTABLE 状态下调用 GC_push_roots 扫描根节点

![](./T2mfb3j7doJovOxg6oScqbLGnUd.png)

GC_push_all_stack_section 和 GC_push_all_register_sectons 方法分别扫描栈和寄存器，最终都是通过 GC_push_all_eager 进行扫描

![](./L3pvbcwY1oyLt3xSDWjc2v49n6e.png)

扫描过中判断内存块地址是否是有效的，被引用的托管堆内存节点有以下逻辑：

1.是否在大致范围内 GC_least_plausible_heap_addr～ GC_greatest_plausible_heap_addr。如果在这个范围内，再调用 GC_mark_and_push_stack 方法进一步判断。

2.判断是否能取到 header 信息，因为每个 hblk 内存块都对应一个 header 信息，能取到说明地址位于 hblk 内存块中。

3.判断位于的 hblk 内存块是否空闲，不空闲说明这个内存块被分配给上层使用。

![](./Vguybej7Co9aaOxCNRacZol3nQf.png)

source 对应的是根节点，p 对应的是根节点的地址。通过 GET_HDR 取出地址 p 对应的 Header 信息，如果能够找到对应 Header 信息 ，说明地址 p 在一个有效的 hblk 内存块中，然后进一步判断这个内存块是否空闲，最后在将其添加到临时数组中。

在 PUSH_CONTENTS_HDR 方法中计算内存节点位于 hblk 内存块中的偏移值，然后把偏移值当做 hb_marsk 数组的下标去修改内存的标记状态，再将增加整体引用个数 hb_n_marks。

![](./Q4lHbVKrMoxCU3xAw6ocrNGpnlg.png)

接着调用 PUSH_OBJ，将引用的内存节点地址加入一个临时数组中，作为接下来的递归扫描的起点。mark_stack_top 是一个数组，存放的 obj 是刚才扫描出的被引用的托管堆内存节点地址。

![](./ZnXhb983CoJPatxkKK4cGkUqnpf.png)

#### 堆内存递归扫描

遍历完根对象后接下来开始遍历临时数组 mark_stck_top,  扫描被引用托管堆中的地址值，判断这个地址是否还在托管堆中。具体逻辑是通过 MARK_FROM_MARK_STACK 宏调用 GC_mark_from 方法。

![](./Z1trb3Oyzo1NW7xmn5TcQ8KVnCc.png)

current_p 是当前内存地址指针， current 是 current_p 所指向的内存地址。然后调用 PUSH_CONTENTS 方法判断 current 是不是托管堆中的内存节点，接着 current_p += ALIGEMENT,逐渐偏移指针直到内存空间全部扫描完。

![](./DqSBbzXYwomjRmxy7mucPbOlnRh.png)

判断内存是否引用逻辑依旧是查找 Header 信息，调用 HC_GET_HDR 方法获取 current 所在的 hblk 内存块对应的 header 信息，和 GET_HDR 方法不同的是，HC_GET_HDR 方法会借助缓存获取 header 信息，如果有直接返回 header，如果没有，调用 HEADER_CACHE_MISS 方法获取 header 信息，然后存入缓存。

![](./Agy4b5VBRoA7scx5EFvc49fDnye.png)

获取到 header 信息后，调用 PUSH_CONTENTS_HDR 方法如上文所述，修改对应内存节点的标记位，递增 hb_n_marks 整体引用内存块的个数，最后在将这个内存节点加入临时数组 mark_stack_top 中，然后继续遍历 mark_stack_top 数组直到托管堆上所有的内存全部扫描过。

#### 线程恢复

扫描完成后，调用 START_WORLD()重新恢复线程，START_WORLD 会调用 GC_start_world 方法

![](./FGbdbgqZ9opVNQxeyeBcElmQnTb.png)

和 GC_suspend_all 方法对应，除了当前线程（以及已结束、阻塞）之外的线程进行恢复，调用 RAISE_SIGNAL 发送一个恢复的信号。

## Reclaim 阶段

扫描标记完成后，开始对未引用的内存节点进行回收，逻辑在 GC_finish_collection 中触发

![](./WQFFbJYCuoJTVVxwnWXcnp9kn8f.png)

方法中首先判断是否需要检测内存泄露，如果是并检测到了内存泄露，不会触发实现回收。紧接着开始清空 ok_freelist 链表，将链表中的内存节点标记位重置为 0。最后 才开始实际回收。

### GC_start_reclaim

GC_start_reclaim 方法中执行更具体的回收逻辑

![](./ULQYbkvrqotHdTxgtTDcLbPOnzr.png)

首先将 ok_freelist 链表中的内存节点清空，因为对于小内存节点（小于 2024 字节）在释放的时候会加入到 ok_freelist 链表中方便后续内存分配，所以在回收的时候需要先清空一下。

然后接着清空 ok_reclaim_list 链表，ok_reclaim_list 是 Header 结构中的字段存储的是需要回收的对象。

然后调用 GC_apply_to_all_blocks 方法遍历托管堆中所有的 hblk 内存块，对于每个 hblk 内存块调用 GC_reclaim_block，大内存会直接回收，小内存会暂时加入到 ok_reclaim_list 链表中。

最后调用 GC_reclaim_all 方法回收小内存。

### GC_reclaim_block

![](./UVqqbkyG1orBIWxAUiqc7zz9nfb.png)

首先通过 sz 获取 hhdr 内存块的大小，然后将大于 MAXOBJBYTES 的大内存和小于 MAXOBJBYTES 的小内存分开处理。

#### 大内存

大内存一般是由 1~N 个 hblk 内存块组成，而小内存是一个 hblk 分成若干个内存节点，因此 Header 中 hb_marks 数组只存储了一个标记位。调用 mark_bit_from_hdr 获取 header -> hb_marks[0]的值，如果值为 0 表示大内存没有被引用，调用 GC_freehblk 方法将大内存加入到 GC_hblkfreelist 链表中，并且将大内存中的数据全部清空。

#### 小内存

对于小内存节点，它们只是 hblk 内存块的一部分，需要分情况处理 hblk 内存块。

情况 1： hblk 内存块全部为空，调用 GC_block_empty 判断是否 hblk 为空，如果是直接将整块 hblk 添加到 hblk_freelist 链表中，并清空数据。

情况 2： hblk 内存块大部分为空，调用 GC_block_nealy_full 方法判断 hblk 内存块中的节点是否大多数被引用，如果不是就将这个 hblk 内存块加入到待回收链表 ok_reclaim_list 中。

情况 3： hblk 内存块中大部分不为空，如果 hblk 中大部分内存节点被引用，该 hblk 内存块就不会加入都 ok_reclaim_list 中

### GC_reclaim_all

GC_reclaim_all 是最后用于回收小内存的。

![](./Y035b8Sbiof2WYxRdYGcPh13nMg.png)

首先遍历所有 ok_reclaim_list 链表，取出 hblk 内存块，再通过 hblk 内存块找到对应的 header 信息，然后调用 GC_reclaim_small_nonempty_blaock 方法处理，方法中有两个参数第一个参数表示 hblk 内存地址，第二个参数 report_if_found 表示是否仅发现并不回收，这里传入 false 表示需要回收。

![](./TU0kbYSBFoEGqpxuHxbc72YInOg.png)

回收过程需要调用 GC_reclaim_generic 方法，GC_reclaim_generic 方法内存调用 GC_reclaim_clear 方法回收。

![](./G8F6b9bD4oT4dQx6C9lcDVy9n7e.png)

在 GC_reclaim_clear 方法中会遍历 hblk 内存中的所有内存节点，通过 mark_bit_form_hdr 判断内存节点是否被引用，也就是查找其标记位是否为 1，如果是 1 就不需要回收，不是 1 就将其加入到 list 中，也就是 ok_freelist 链表中。到此就完成了所有内存节点的回收。

## 总结

本文主要介绍了 BoehmGC 内存回收中标记和回收这两个关键过程。在标记阶段，先遍历所有内存块将其标记位和引用个数全部清 0，然后挂起线程从根节点开始扫描判断其指向的内存地址是否位于托管堆中，如果是则将其内存块标记起来并递增引用个数，并将其添加到临时数组 mark_stack_top 数组中，然后遍历 mark_stack_top 数组中的托管堆内存块，判断其内存块中的地址值是否还指向托管堆中，如果是就将其添加到临时数组 mark_stack_top 中。扫描标记介绍后就进入了回收阶段，先会判断是否需要检测内存泄露，并标记未被引用的内存块但并不回收，然后清空 ok_freelist 链表，再通过 GC_start_reclaim 开始回收，在回收的过程中会将大小内存按不同的回收策略回收。对于大内存直接将其加入 hblk_freelist 链表中，而对于小内存需要分情况判断，是添加到 hblk_freelist 链表中 还是添加到 ok_reclaim_list 中。
