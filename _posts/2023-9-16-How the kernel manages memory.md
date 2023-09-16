---
layout: post
title: 内核如何管理用户内存
date:  2023-9-16
tags: [cs]
comments: true
author: kevin
---

在检查进程的虚拟地址布局后，我们转向内核及其管理用户内存的机制。

![Linux kernel mm_struct](https://raw.githubusercontent.com/Promin3/Promin3.github.io/main/images/mm_struct.png)

Linux进程作为进程描述符task_struct的实例在内核中实现。Task_struct中的mm字段指向内存描述符mm_struct，这是程序内存的执行摘要。它存储上图所示的内存段的开始和结束、进程使用的物理内存页面数量（rss代表常驻集大小）、使用的虚拟地址空间量以及其他花絮。在内存描述符中，我们还找到了管理程序内存的一组虚拟内存区域和页表。Gonzo的记忆区域如下所示：

![Kernel memory descriptor and memory areas](https://raw.githubusercontent.com/Promin3/Promin3.github.io/main/images/memoryDescriptorAndMemoryAreas.png)

每个虚拟内存区域（VMA）都是一个连续的虚拟地址范围；这些区域永远不会重叠。Vm_area_struct的实例完全描述了内存区域，包括其开始和结束地址，用于确定访问权限和行为的标志，以及用于指定区域映射哪个文件的vm_file字段（如果有的话）。不映射文件的VMA是匿名的。除内存映射段外，上面的每个内存段（例如堆、堆栈）都对应于单个VMA。尽管这在x86机器中很常见，但这并不是被要求的。

程序的VMA存储在其内存描述符中，既作为mmap字段中的链接列表，按启动虚拟地址排序，也作为植根于mm_rb字段的红黑树。红黑树允许内核快速搜索覆盖给定虚拟地址的内存区域。当您读取文件/proc/pid_of_process/maps时，内核只是浏览进程的VMA链接列表并打印每个VMA。

在Windows中，EPROCESS块大致是task_struct和mm_struct的混合。Windows中类似于VMA是虚拟地址描述符（VAD）；它们存储在AVL树中。

4GB虚拟地址空间被分为页面。32位模式的x86处理器支持4KB、2MB和4MB的页面大小。Linux和Windows都使用4KB页面映射虚拟地址空间的用户部分。字节0-4095落在第0页，字节4096-8191落在第1页，以此类。VMA的大小必须是页面大小的倍数。以下是4KB页面中的3GB用户空间：

![4KB Pages Virtual User Space](https://raw.githubusercontent.com/Promin3/Promin3.github.io/main/images/pagedVirtualSpace.png)

处理器查阅页表，将虚拟地址转换为物理内存地址。每个进程都有自己的一组页表；每当发生进程切换时，用户空间的页表也会被切换。Linux在内存描述符的pgd字段中存储一个指向进程页表的指针。对于每个虚拟页面，页面表中都对应一个页面表条目（PTE），在常规x86分页中，这是一个简单的4字节记录，如下所示：

![x86 Page Table Entry (PTE) for 4KB page](https://raw.githubusercontent.com/Promin3/Promin3.github.io/main/images/x86PageTableEntry4KB.png)

Linux具有读取和设置PTE中每个标志的功能。位P告诉处理器虚拟页面是否存在于物理内存中。如果清除（等于0），访问页面会触发页面错误。请记住，当此位为零时，内核可以随心所欲地处理其余字段。R/W标志代表读/写；如果清除，则页面为只读。Flag U/S代表user/supervisor；如果清除，则只能由内核访问该页面。这些标志用于实现我们之前看到的只读内存和受保护的内核空间。

位D和A用于dirty和accessed。dirty page有写入，而accessed page有写入或读取。两个标志都是粘性的：处理器只设置它们，它们必须由内核清除。最后，PTE存储与此页面对应的起始物理地址，与4KB对齐。这个naive-looking字段是一些痛苦的根源，因为它将可寻址的物理内存限制在4 GB。其他PTE字段是留作未来使用，物理地址扩展也是如此。

虚拟页面是内存保护的单元，因为它的所有字节都共享U/S和R/W标志。然而，相同的物理内存可以由不同的页面映射，可能使用不同的保护标志。请注意，在PTE中看不到执行权限。这就是为什么经典的x86分页允许在堆栈上执行代码，从而更容易利用堆栈缓冲区溢出（仍然可以使用return-to-libc和其他技术利用不可执行的堆栈）。缺乏PTE禁止执行标志说明了一个更广泛的事实：VMA中的权限标志可能会、也可能不会干净地转化为硬件保护。内核会做它能做的事，但最终架构限制了可能的事情。

虚拟内存不存储任何东西，它只是将程序的地址空间映射到底层物理内存，处理器将其作为称为物理地址空间的大块访问。虽然总线上的内存操作有点相关，但我们可以在这里忽略这一点，并假设物理地址以一字节的增量从零到可用内存的顶部。这个物理地址空间被内核分解为页面帧（page frame）。处理器不知道或不关心帧，但它们对内核至关重要，因为页面帧是物理内存管理的单元。Linux和Windows都在32位模式下使用4KB页面帧；以下是具有2GB内存 (RAM) 的机器示例：

![Physical Address Space](https://raw.githubusercontent.com/Promin3/Promin3.github.io/main/images/physicalAddressSpace.png)

在Linux中，每个页面帧都被一个描述符和几个标志track。这些描述符一起跟踪计算机中的整个物理内存；每个页面帧的精确状态总是已知的。物理内存使用伙伴内存分配技术进行管理，因此，如果可以通过伙伴系统分配，则页面帧是free的。分配的页面帧可能是匿名的，保存程序数据，也可能在页面缓存中，保存存储在文件或块设备中的数据。Windows有一个类似的页面帧号（PFN）数据库来跟踪物理内存。

![Physical Address Space](https://raw.githubusercontent.com/Promin3/Promin3.github.io/main/images/heapMapped.png)

蓝色矩形表示VMA范围内的页面，而箭头表示将页面映射到页面帧的 page table entries 。一些虚拟页面缺少箭头；这意味着其相应的PTE具有清晰的当前标志。这可能是因为页面从未被触碰过，或者因为它们的内容被替换掉了。无论哪种情况，访问这些页面都会导致页面错误，即使它们在VMA中。VMA和页表不一致似乎很奇怪，但这种情况经常发生。

VMA就像程序和内核之间的合同。您要求做一些事情（内存分配、文件映射等），内核会说“确定”，然后创建或更新适当的VMA。但它实际上并没有立即满足请求，它会等到页面错误发生时才完成实际工作。The kernel is a lazy, deceitful sack of scum; 这是虚拟记忆的基本原理。它适用于大多数情况，有些是熟悉的，有些是令人惊讶的，但规则是VMA记录已商定的内容，而PTE反映了懒惰内核实际所做的工作。这两个数据结构一起管理程序的内存；两者都在解决页面故障、释放内存、交换内存等方面发挥着作用。让我们以内存分配的简单情况为例：

![Example of demand paging and memory allocation](https://raw.githubusercontent.com/Promin3/Promin3.github.io/main/images/heapAllocation.png)

当程序通过brk()系统调用要求更多内存时，内核只需更新堆VMA并调用它。此时实际上没有分配页面帧，新页面也不存在于物理内存中。一旦程序尝试访问页面，就会调用处理器页面故障和do_page_fault()。它使用find_vma()搜索覆盖故障虚拟地址的VMA。如果找到，也会对照尝试访问（读取或写入）检查VMA的权限。如果没有合适的VMA，则没有合同涵盖尝试的内存访问，该过程将受到分段故障的惩罚。

当发现VMA时，内核必须通过查看PTE内容和VMA类型来处理故障。在我们的案例中，PTE显示页面不存在。事实上，我们的PTE是完全空白的（都是零），这在Linux中意味着虚拟页面从未被映射过。由于这是一个匿名VMA，我们有一个纯粹的RAM事件，必须由do_anonymous_page()处理，它分配一个页面帧，并制作一个PTE将有故障的虚拟页面映射到新分配的帧上。

事情本可以有所不同。例如，交换页面的PTE在当前标志中为0，但不是空白的。相反，它存储了保存页面内容的交换位置，该位置必须从磁盘读取并由do_swap_page()加载到页面帧中，称为重大故障。

我们通过内核的用户内存管理之旅的前半部分到此结束。在下一篇文章中，我们将将文件放入组合中，以构建内存基础知识的完整图景，包括对性能的后果。

[翻译自该文章](https://manybutfinite.com/post/how-the-kernel-manages-your-memory/)
