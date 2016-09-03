---
layout: post
title: slow memory access on Xen guest
author: JamesW
categories: Kernel Virtualization
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

## 问题: diskdump某段内存异常缓慢

最近移植diskdump到XEN的native block driver (xen-blkfront.c)上, 本来一切都好, 可是把guest VM的内存调到4GB以上的时候, diskdump却一直不能成功. 调试之后发现diskdump在dump某段内存(大约256MB)时异常缓慢, memcpy一个page需要耗费大约一秒的时间, 在虚拟化环境下内存访问变慢是可以预期的, 可是这么慢一定是哪里出了问题.

实验环境如下:

* guest - HVM, Linux kernel v3.2, 6GB memory
* host - XenServer 6.5

在AWS上的测试也是一样, Google没搜到有用的信息, Xen的论坛问了问没人解答, 看啦别人都没有碰到过这个问题, 只好自己解决. 不过有趣的是, 同样的guest在Fedora 24的Xen上却没有问题, 应该host/guest两边都有关系.

## 重现问题: 找到最简测试用例

Diskdump每次都要重启VM, 为了加快调试, 考虑设计一个新的测试用例. 因为diskdump memcpy慢这个结论是确定的, 应该可以通过/dev/mem在触发这个问题:

	for i in {0..2000}; do
		offset=$((4600+i))
		echo -n "$offset "
		dd if=/dev/mem of=/dev/null bs=1M count=1 skip=$offset 2>/dev/null
	done

这个测试用例在我们自己定制的v3.2内核上的确是可以重现这个, 在Fedora guest上怎么也不能重现问题, 本来还以为找到方向了, 比较自己的内核代码和Fedora的内核代码即可, 虽然版本相差不少. 不过可惜的是, 最后发现这个脚本之所以在Fedora能够很快完成, 是因为Fedora的kernel config已经enable CONFIG_STRICT_DEVMEM, 也就是说不能通过/dev/mem访问[绝大部分]主存, 只不过因为把stderr重定向到/dev/null, 所以忽视了这个问题. 在测试的开始阶段, rootcause还没有完成的时候, 忽视一些错误的确不是best practice, 不怕信息过多, 就怕忽视了某些信息. 所以改成这样:

	for i in {0..2000}; do
		offset=$((4600+i))
		echo -n "$offset "
		dd if=/dev/mem of=/dev/null bs=1M count=1 skip=$offset
	done

这个脚本可以重现问题, 可是运行以后, guest马上变得异常缓慢, 即使最后kill了这个脚本之后, 系统还是不能正常响应. 因为已经知道一个page的访存都是秒级的, 而且出问题的内存地址还是相对连续的, 最后修改如下, 也能很好地重现问题, 并且系统在kill该脚本之后还能够正常运行: 

	for i in {0..2000}; do
		offset=$((4600+i))
		skip_pages=$((offset * 256))
		echo -n "$offset "
		t1=`date +%s`
		dd if=/dev/mem of=/dev/null bs=4k count=1 skip=$skip_pages
		t2=`date +%s`
		echo $((t2-t1))
	done

## 分析问题: 找出异同

测试用例出来之后, 就可以进行更多的调试了. 可惜的是, 我们用的XenServer是免费试用版, 不要说源码, 很多工具都没有, dom0上的调试手段相当有限.

* 当在guest运行脚本的时候, dom0上可以观测到该VM对应的qemu-dm CPU占用率马上升到100%.
* strace qemu-dm这个进程, 就是在不停重复这个过程, fd=6指向/proc/xen/privcmd

		mmap(NULL, 1048576, PROT_READ|PROT_WRITE, MAP_SHARED, 6, 0) = 0x7f27aa749000
		ioctl(6, IOCTL_PRIVCMD_MMAPBATCH_V2, ->{num=256,dom=1127,addr=0x7f27aa749000,arr=[1390848 to 1391103],err=[-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22,-22]}) = 0
		munmap(0x7f27aa749000, 1048576)         = 0

根据已有的知识, 可以确定的是: 

* 内存虚拟化理应不会涉及到qemu, 只有在模拟设备的时候才会用到qemu.
* 导致内存访问慢的地方的确是RAM空间, 这个可以通过/proc/iomem确认.

可是这个问题就是出现在RAM空间, 并且内存访问导致qemu的CPU使用率上升到100%. 由于上面的测试脚本可以很容易定位到出问题的地址, 所以尝试读出一个page的内容看看:

	offset=5409; skip_pages=$((offset * 256)); dd if=/dev/mem of=/tmp/abc bs=4k count=1 skip=$skip_pages

读出的所有内容皆为0xff, 再尝试写的效果:

	offset=5409; skip_pages=$((offset * 256))
	dd if=/dev/zero of=/dev/mem bs=4k count=1 seek=$skip_pages
	dd if=/dev/mem of=/tmp/abc bs=4k count=1 skip=$skip_pages

即使写0之后, 读出的依然是0xff. 这个倒真有点像设备内存, 不过这个可能性已经排除. 还有一个测试值得一试, 也就是Ubuntu系统自带的memtest+, 它完全独立于Linux运行, 如果它没有问题的话, 那至少说明Linux这边做了一些特殊的操作, 也就说并不需要改变Hypervisorl那边的行为就可以解决这个问题. 当然上面已经讲过, Hypervisor那边的改变也能解决这个问题, 不过Hypervisor往往不是我们能够控制的. 结果memtest+并没有发现问题, 因为memtest+不只是读而已, 还包括写入验证等等, 所以可以确定memtest+并没有出现上面不能写入的问题.

那么同样的内存(虚拟化的), Linux kernel到底做了什么导致这个问题?

首先需要看看虚拟机上的内存跟物理机上到底有哪些不同, 特别是HVM(或者PVHVM)的guest上, 内存虚拟化是借助硬件来实现的, 而物理内存分布是通过模拟的e820传递进来, 这些都不太可能引起如果巨大的性能损失, 并且写入跟读取的内容不一样, 这就更不可能了. 那么剩下的一个可能性就是Balloon driver.  

Balloon driver寄生在guest OS里面, 在"适当"的时候会reserve (alloc_page) 一些guest的内存, 并释放给Host, 从而Host可以把这部分内存分配给其他的guest, 当Host上的内存支持不了所有guest的target内存的时候喔, 这也不失为一种办法. 总之, Balloon提供了一种机制来动态调整guest的真实内存大小.

可以肯定的是, 当内存被Balloon reserve之后, 操作系统就不应该再使用这部分内存, 由于Balloon是通过alloc_page来实现的, 所以一定程度上是有保证的, Linux kernel不会把这些内存再分配出去. 不过我们这里是直接访问/dev/mem (diskdump类似, 直接遍历e820表), 所以这些pages还是可见的, 这也不是什么大问题, 问题是访问的时候(read/write)为什么会这么慢? 为什么Hypervisor不能够直接或者更快的处理, 而需要使用到qemu? qemu又在做什么呢?

Disable Balloon之后, 这个问题果然消失. 所以解决办法就很简单:

* disable balloon即可
* 如果需要保留balloon, 那么可以在balloon reserve内存的时候, 把相应的page做个标记,
  在diskdump的时候跳过. 当然如果想支持/dev/mem, 也需要相应的调整.

## 回顾问题

* 为什么每次都能碰到这个问题? 按理说Balloon并不总是reserve内存的?

  v3.2的Linux kernel在Xen虚拟化的支持还有很多不足和bug, 比如Balloon driver, 在初始化的时候
  就会reserve一部分(大约256MB)内存, upstream commit c275a57f解决了这个问题:

	diff --git a/drivers/xen/balloon.c b/drivers/xen/balloon.c
	index b232908..1b62304 100644
	--- a/drivers/xen/balloon.c
	+++ b/drivers/xen/balloon.c
	@@ -641,7 +641,7 @@ static int __init balloon_init(void)

		balloon_stats.current_pages = xen_pv_domain()
			? min(xen_start_info->nr_pages - xen_released_pages, max_pfn)
	-               : max_pfn;
	+               : get_num_physpages();
		balloon_stats.target_pages  = balloon_stats.current_pages;
		balloon_stats.balloon_low   = 0;
		balloon_stats.balloon_high  = 0;

  可以看出, 这256MB内存基本上就是由于PCI Bus上iomem导致的. 如果使用新的Kernel guest, 这个问题还真变得不一样.

* 为什么总是出现在>4GB内存的guest?

  如上, 在小内存系统max_pfn不会包括PCI内存, 也可以看出4GB并不是边界, 而是0xf0000000附近.

	-bash-3.00# cat /proc/iomem
	[...]
	f0000000-fbffffff : PCI Bus 0000:00
	  f0000000-f1ffffff : 0000:00:02.0
	  f2000000-f2ffffff : 0000:00:03.0
	    f2000000-f2ffffff : xen-platform-pci
	  f3000000-f3000fff : 0000:00:02.0
	fc000000-ffffffff : reserved
	  fec00000-fec003ff : IOAPIC 0
	  fed00000-fed003ff : HPET 0
	  fee00000-fee00fff : Local APIC
	[...]

* 访问Balloon reserved内存到底会触发什么操作?

  之前提到过, 在Fedora 24的Xen上已经没有这个问题了. 只要Hypervisor维护了这个信息, 应该是
  很容易实现的. 具体代码暂时没有深入.
