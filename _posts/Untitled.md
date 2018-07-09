内存管理一直是操作系统几大核心内容之一，线程调度，文件加载，事件处理等操作都是建立在内存管理的基础之上。现代操作系统为了达到适配各种硬件架构，方便上层操作等目的都会引入虚拟内存的概念，本文主要介绍虚拟内存在Darwin中是如何实现的。


# Mach下虚拟内存结构

       Mach提供了一套非常全面的底层虚拟内存机制，提供给诸如BSD层的_Malloc()，可执行文件加载等等模块使用。下面话不多说直接上图看一下整体结构并且分具体模块解释（为了方便理解只保留的结构相关的部分，省略了大部分属性变量，完整的类定义可以在vm\_map.h等文件中查看）

![虚拟内存结构]({{site.url}}/gitblog/assets/img_posts/2/1.png)


## vm\_map

        处于龙头老大低位，宏观上看就是代表了若干段地址空间的映射，它地址空间是由一个`vm_map_entry`的双向列表组成（按照地址排序）。其中的`pmap`属性是用于映射到具体的物理地址的，会依据硬件的变化而变化。在上一篇讨论mach-o文件加载的过程中，`load_machfile`有如下代码：

```c
	pmap = pmap_create(get_task_ledger(ledger_task),
			   (vm_map_size_t) 0,
			   result->is64bit);
	map = vm_map_create(pmap,
			0,
			vm_compute_max_offset(result->is64bit),
			TRUE);
```
       这里先创建了一个pmap然后根据pmap创建了vm_map，该vm_map对象后续被用于加载各个段，启动进程等，代表了整个进程的地址空间，并且虚拟地址的最大offset设为了64位/32位系统下最大的寻址空间。

## vm\_map\_entry

        `vm_map`所持有的链路中的一个节点，entry顾名思义代表一段内存的入口，并且这里的内存是一段连续的虚拟内存。内存的权限控制信息（protection信息，例如r/w/x等对内存页的操作）也存储在该部分。一个`vm_map_entry`持有一个`vm_map_object`的union，它通常指向一个`vm_object`的双向链表，也可以是一个子的`vm_map`。在mach-o文件加载过程中的`map_segment`函数中，有如下代码：

			ret = vm_map_enter_mem_object(
				map,
				&cur_start,
				cur_end - cur_start,
				(mach_vm_offset_t)0,
				VM_FLAGS_FIXED,
				cur_vmk_flags,
				VM_KERN_MEMORY_NONE,
				IPC_PORT_NULL,
				0, /* offset */
				TRUE, /* copy */
				initprot, maxprot,
				VM_INHERIT_DEFAULT);

       这里`load_segment`的过程就是将mach-o文件中的段映射到虚拟内存中去，这里将每一段分成了一个`vm_map_entry`，由于`vm_map_entry`内的地址是连续的，通过随机偏移量加上段本身大小等信息，达到了既保证段内相对位置不变，又可以把各个段随机映射到不同位置的目的。

## vm\_object

        代表`vm_map_entry`指向具体的内存，他主要是由一个`vm_page`的双向链表(`memq`)，一个负责内存swap等操作的pager（下面详细介绍）和一些其他的属性标识等组成。

### pager

       由于实际的物理内存数量是有限的，如果所需内存超过RAM中可用内存的数量，就需要将内存暂存到列如磁盘，文件，设备等处，用到时再拿回来，达到用户态无感知循环使用内存的目的，这就是内存swap操作。每个操作系统基本都会有一套自己的swap机制，Mach的内存swap是由pager实现的。

       首先pager是一个`memory_object`对象，定义如下：

```c
/*
 * "memory_object" and "memory_object_control" types used to be Mach ports
 * in user space and can be passed as such to some kernel APIs.
 * Their first field must match the "io_bits" field of a
 * "struct ipc_object" to identify them as a "IKOT_MEMORY_OBJECT" and
 * "IKOT_MEM_OBJ_CONTROL" respectively.
 */
typedef struct 		memory_object {
	mo_ipc_object_bits_t			mo_ikot; /* DO NOT CHANGE */
	const struct memory_object_pager_ops	*mo_pager_ops;
	struct memory_object_control		*mo_control;
} *memory_object_t;
```

       它是一个可以被用于在用户空间与硬件端口通信的标准。在Mach中，常见的分页器有Default Pager(默认分页器)，VNode Pager(用与内存与文件的映射)，Device Pager（设备）等。所有的分页器都实现了一系列标准接口，其中最重要的就是`page_in`和`page_out`。

```c
/*
 *	Request data from this memory object.  At least
 *	the specified data should be returned with at
 *	least the specified access permitted.
 *
 *	[Response should be upl commit over the specified range.]
 */
routine	memory_object_data_request(
		memory_object		: memory_object_t;
		offset			: memory_object_offset_t;
		length			: memory_object_cluster_size_t;
		desired_access		: vm_prot_t;
		fault_info		: memory_object_fault_info_t);
        处理page_in请求，也就是从后备存储读页进来，读到从memory_object的offset为起始，offset_length为终点的区域。
```

```c
/*
 *	Return data to manager.  This call is used in place of data_write
 *	for objects initialized by object_ready instead of set_attributes.
 *	This call indicates whether the returned data is dirty and whether
 *	the kernel kept a copy.  Precious data remains precious if the
 *	kernel keeps a copy.  The indication that the kernel kept a copy
 *	is only a hint if the data is not precious; the cleaned copy may
 *	be discarded without further notifying the manager.
 *
 *	[response should be a upl_commit over the range specified]
 */
routine   memory_object_data_return(
		memory_object		: memory_object_t;
		offset		 	: memory_object_offset_t;
		size			: memory_object_cluster_size_t;
	out	resid_offset	 	: memory_object_offset_t;
	out	io_error		: int;
		dirty			: boolean_t;
		kernel_copy		: boolean_t;
		upl_flags		: int);
```

        处理`page_out`请求，也就是把页存到后备存储中去，也是从`memory_object`的offset为起始，offset_length为终点的区域，将数据拷贝到后备存储中，并且将其标记为可用。

        在mach-o文件加载过程中，就是将文件内容作为一个`vnode`，然后利用`page_in`映射能力把文件中的不同段，签名等内容加载到内存中去。

![vnode]({{site.url}}/gitblog/assets/img_posts/2/2.png)

## vm_page

       真正映射到物理页的部分，一个页可以有多种状态：驻留，交换到后备存储，被加密，clean，dirty等。这里要注意的是，这里一个vm_page_t的大小是64个字节，这是因为在arm64和x86_64架构下，一个cache line的大小正好是64字节，在内存对齐的前提下，方便CPU一次性读取一个页。（关于CPU cache line的简单介绍 <https://stackoverflow.com/questions/3928995/how-do-cache-lines-work>，CPU各级缓存策略详细介绍和示例：<http://igoro.com/archive/gallery-of-processor-cache-effects/>）

## pmap

       pmap是`vm_map`持有的真正将虚拟内存映射到物理内存的模块，从逻辑设计的角度上来讲，它可以分为两个部分：硬件无关层和硬件相关层。

### 硬件无关层

       从实际实现上来讲，所谓的硬件无关层其实并没有做什么事情，但是这部分定义了与上层交互的硬件无关的标准API（详细可以在<vm/pmap.h>中找到）。这里定义了诸如`pmap_create`，`pmap_enter`等创建和管理内存映射的API，上层可以不需要关心具体的硬件实现，依照标准来处理内存。

### 硬件相关层

      这里是具体实现虚拟地址映射的地方，在苹果的Darwin源码中，具体实现了arm，i386，x86\_64等硬件架构的pmap。每个硬件架构都有一套自己的虚拟地址到物理地址的映射方式，但是都无外乎通过页表，PTE（page table entry）等完成地址映射。下面以arm为例，简单描述下虚拟地址到物理地址的映射过程。

      首先，虚拟地址中内核空间（kernal space，只被操作系统内核使用）和用户空间（比如运行app用到的内存）是完全分离的，如下图所示：

![用户态内核态分离]({{site.url}}/gitblog/assets/img_posts/2/3.png)

       这里内核空间和用户空间的基地址是在Translation Table Base Registers (TTBR0\_EL1) 和(TTBR1\_EL1)中指定的，比如说图例中用户空间可以访问0x0 到 0x0000FFFF\_FFFFFFFF，内核空间可以访问0xFFFF0000\_00000000 到 0xFFFFFFFF\_FFFFFFFF 的虚拟地址空间，也就是前16位必须是0或者1，否则就会触发一个地址错误。

       在此基础之上，假设我们要映射一个42bit的虚拟地址空间到48bit的物理地址页（其实armV6支持的是64KB和4KB两种大小的页），流程如下：

![映射过程]({{site.url}}/gitblog/assets/img_posts/2/4.png)

这里整体流程如下：

1. 首先看虚拟地址的第63到42位，当它们全等于0的时候，说明需要使用TTBR0为基地址的虚拟地址转换表。
2. 根据第41到29位的数值作为表内偏移量，找到表内对应的具体PTE（page table entry），然后从改项中读出物理页的基地址。
3. 将从PTE中读出的基地址与剩下的第28到0位拼接，成为真正的具体物理地址。

        这里只是最简单的一个转换表的情况，通常会有多个转换表，也就是前一个读出的PTE作为下一张表的基地址，然后加上虚拟地址的最后一段继续拆分的偏移量来定位新的PTE的做法，达到段页分离等目的。

参考文献

【1】Darwin-XNU源码：https://github.com/apple/darwin-xnu

【2】Mac OS® X and iOS Internals： http://newosxbook.com/MOXiI.pdf

【3】arm手册：https://www.scss.tcd.ie/~waldroj/3d1/arm_arm.pdf