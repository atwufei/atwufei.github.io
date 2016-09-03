---
layout: post
title: scatterlist in xen-blkfront
author: JamesW
categories: Kernel Virtualization
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

## Persistent grants in xvd

xen-blkfront driver (xvd) 对性能做了一系列优化, 其中主要的一个优化就是使用persistent grants代替了传统map/unmap的grant table, 引入了一次额外的内存拷贝, 但是减少了map/unmap的开销, 特别是在多CPU的情况下, unmap导致的TLB flush会极大地影响性能. 甚至在backend不支持persistent grants的情况下, 由于guest只是一次性grant这些pages, 从而减少了backend lock contention, 在多CPU的情况下性能还是能够 得到一定地提升. 也就是说, 在这种情况下, 内存拷贝比零拷贝还快不少. 有关persistent grants的详细信息可以参考 [Improving block protocol scalability with persistent grants](https://blog.xenproject.org/2012/11/23/improving-block-protocol-scalability-with-persistent-grants/)

由于我们用的内核版本是v3.2, 需要backport这个patch. 顺便说一下, v3.2真的不是Xen ready, 还缺少很多feature和bugfix, 不过对于开发人员来说却不见得是什么坏事, 刚好可以更深入的理解这些东西.

backport的过程相对简单, 主要就是upstream commit 0a8704a51, 只是在测试的时候需要验证persistent grants是否前后端都enabled, 否则测试结果可能不是我们想要的. 不过问题就出在这个upstream commit, 竟然parted都会出现错误. 由于我们都是用预先分好区的VMDK或者RAW格式的disk, 正常的读写也都没有什么问题, 所以并没有在第一时间就发现这个问题. 看来好的unit test还是很有必要, 而且当时乐观的信任upstream的commit, 虽然upstream的commits总体水平会好一些, 更多design和review, 但最重要的还是测试, 也是因为upstream的绝大部分代码经过了大量的测试才会让我们相信甚至默认它是稳定的. 不过Xen小众了一点, 测试不充分也是可以预期的, 而且很多feature刚引入的时候有bug也是正常.

## 问题: Parted不能正常使用

错误的具体情况如下:

	-bash-3.00# parted /dev/xvdb mklabel gpt
	-bash-3.00# parted /dev/xvdb p
	Error: Can't have the end before the start!

跟踪parted的源码的话, 可以发现该错误是由于解析GPT的分区表时出错. 可以肯定的是, 除了使用parted之外, 我们并没有故意地去破坏分区表. 而且经过一番测试, 还可以确定的是, 这个bug就是backport persistent grants引入的.

首先熟悉一下 [GPT的磁盘格式](https://en.wikipedia.org/wiki/GUID_Partition_Table)

* LBA 0 - Protective MBR, 0x55aa
* LBA 1 - Primary GPT HEADER, "EFI PART"
* LBA [2, 34) - Partition entries

有了这些基本概念之后, 就一定程度上能够分辨出数据的正确性.

## 重现问题: 设计Unit Test并不总能成功

现在来trace一下parted的具体流程, 为了检查数据的正确性, 特别使用了"-s 4096", 这样strace就能够显示4096的读入数据.

	-bash-3.00# strace -s 4096 -o /tmp/a parted /dev/xvdb p

检查了输出结果, 有趣的事情发生了, parted会多次读取某些offset的sector, 可是结果却并不一样, 而parted的print命令不应该有写操作.

	lseek(3, 512, SEEK_SET)                 = 512
	read(3, "EFI PART\0\0\1\0\\\0\0\0!a^A\0\0\0\0\1\0\0\0\0\0\0\0\377\377?\1\0\0\0\0\"\0\0\0\0\0\0\0\336\377?\1\0\0\0\0\252\222\333\215\v\342SN\260j\224\301\2156\261q\2\0\0\0\0\0\0\0\200\0\0\0\200\0\0\0\206\322T\253\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", 512) = 512
	lseek(3, 1024, SEEK_SET)                = 1024
	read(3, "==ALL 0==", 512) = 512
	lseek(3, 1536, SEEK_SET)                = 1536
	read(3, "==ALL 0==", 512) = 512
	lseek(3, 2048, SEEK_SET)                = 2048
	read(3, "==ALL 0==", 512) = 512

	[...]

	lseek(3, 1024, SEEK_SET)                = 1024
	read(3, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0EFI PART\0\0\1\0\\\0\0\0!a^A\0\0\0\0\1\0\0\0\0\0\0\0\377\377?\1\0\0\0\0\"\0\0\0\0\0\0\0\336\377?\1\0\0\0\0\252\222\333\215\v\342SN\260j\224\301\2156\261q\2\0\0\0\0\0\0\0\200\0\0\0\200\0\0\0\206\322T\253\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0 ...
	0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\1\0\356\376\377\377\1\0\0\0\377\377?\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0U\252"..., 16384) = 16384


因为现在还没有创建分区表, 所以LBA[2, 34)读出零应该是正常的, strace前面的结果也是如此, 不过在后面的一次read的时候在2048B这个位置却读出了"EFI PART"的字样, 而这里的内容跟512B的内容却是一样的.

虽然parted本身在这个问题上并不是特别复杂, 但是如果能够使用更简单的测试用例重现出来, 无疑会更加容易debug. 尝试写了一个简单的测试用例, 主要就是跟strace的结果和blktrace的log, 最后达到的效果就是I/O相关的系统调用一致, 而且blktrace的log也是一模一样. 因为使用strace进行调试, 错误处理也没有必要.

	#define _XOPEN_SOURCE 600
	#define _GNU_SOURCE
	#include <stdlib.h>
	#include <sys/types.h>
	#include <sys/stat.h>
	#include <fcntl.h>
	#include <linux/fs.h>

	int main(int argc, char *argv[])
	{
		int fd;
		char *file;
		void *p, *diobuf;
		int buf_len = 32 * 1024;
		int i;
		int offset;

		file = argv[1];
		offset = atoi(argv[2]) * 512;

		fd = open(file, O_RDWR | O_DIRECT);

		posix_memalign(&p, 4096, buf_len);
		diobuf = p + offset;

		lseek(fd, 0, SEEK_SET); read(fd, diobuf, 512);

		for (i = 0; i < 16; i++) {
			lseek(fd, i * 512, SEEK_SET); read(fd, diobuf, 512);
		}

		lseek(fd, 0, SEEK_SET); read(fd, diobuf, 512);
		lseek(fd, 0, SEEK_SET); read(fd, diobuf, 512);
		lseek(fd, 512, SEEK_SET); read(fd, diobuf, 512);
		lseek(fd, 0, SEEK_SET); read(fd, diobuf, 512);
		lseek(fd, 512, SEEK_SET); read(fd, diobuf, 512);

		lseek(fd, 0, SEEK_SET); read(fd, diobuf, 512);
		lseek(fd, 512, SEEK_SET); read(fd, diobuf, 512);
		lseek(fd, 1024, SEEK_SET); read(fd, diobuf, 16384);

		free(p);
		close(fd);

		return 0;
	}

可惜的是, 即使这样, 这个测试用例也不能重现问题, 看来不[只]跟I/O的大小顺序相关.

## 分析问题: 聚焦改变

考虑到persistent grants主要是在guest引入了一次额外的拷贝, 所以猜想是这次拷贝出了问题. 当然这也可以通过dom0来侧面验证, 如果dom0读出了正确的数据, 而guest却没有, 那guest的嫌疑就更大了.

再来看看这次拷贝的具体代码:

	-static void blkif_completion(struct blk_shadow *s)
	+static void blkif_completion(struct blk_shadow *s, struct blkfront_info *info,
	+                            struct blkif_response *bret)
	 {
		int i;
	-       for (i = 0; i < s->req.nr_segments; i++)
	-               gnttab_end_foreign_access(s->req.u.rw.seg[i].gref, 0, 0UL);
	+       struct bio_vec *bvec;
	+       struct req_iterator iter;
	+       unsigned long flags;
	+       char *bvec_data;
	+       void *shared_data;
	+       unsigned int offset = 0;
	+
	+       if (bret->operation == BLKIF_OP_READ) {
	+               /*
	+                * Copy the data received from the backend into the bvec.
	+                * Since bv_offset can be different than 0, and bv_len different
	+                * than PAGE_SIZE, we have to keep track of the current offset,
	+                * to be sure we are copying the data from the right shared page.
	+                */
	+               rq_for_each_segment(bvec, s->request, iter) {
	+                       BUG_ON((bvec->bv_offset + bvec->bv_len) > PAGE_SIZE);
	+                       i = offset >> PAGE_SHIFT;
	+                       shared_data = kmap_atomic(
	+                               pfn_to_page(s->grants_used[i]->pfn));
	+                       bvec_data = bvec_kmap_irq(bvec, &flags);
	+                       memcpy(bvec_data, shared_data + bvec->bv_offset,
	+                               bvec->bv_len);
	+                       bvec_kunmap_irq(bvec_data, &flags);
	+                       kunmap_atomic(shared_data);
	+                       offset += bvec->bv_len;
	+               }
	+       }
	+       /* Add the persistent grant into the list of free grants */
	+       for (i = 0; i < s->req.nr_segments; i++) {
	+               llist_add(&s->grants_used[i]->node, &info->persistent_gnts);
	+               info->persistent_gnts_c++;
	+       }
	 }

虽然看起来怪怪的, 但是本人并不是专门做I/O stack的 (我觉得这个作者应该也不是:), 所以一时半会也没能找到一个反例. 但是想想既然这么严重的问题, 而这也不是太新的feature, 很可能已经有了bugfix. 搜索了一下blkif_completion相应的commits, 果不其然, 找到upstream commit d62f69185, log里面还给出了一个触发条件:

	1st loop in rq_for_each_segment
	 * bv_offset: 3584
	 * bv_len: 512
	 * offset += bv_len
	 * i: 0

	2nd loop:
	 * bv_offset: 0
	 * bv_len: 512
	 * i: 0

	diff --git a/drivers/block/xen-blkfront.c b/drivers/block/xen-blkfront.c
	index cfdb033..11043c1 100644
	--- a/drivers/block/xen-blkfront.c
	+++ b/drivers/block/xen-blkfront.c
	@@ -836,7 +836,7 @@ static void blkif_free(struct blkfront_info *info, int suspend)
	 static void blkif_completion(struct blk_shadow *s, struct blkfront_info *info,
				     struct blkif_response *bret)
	 {
	-       int i;
	+       int i = 0;
		struct bio_vec *bvec;
		struct req_iterator iter;
		unsigned long flags;
	@@ -853,7 +853,8 @@ static void blkif_completion(struct blk_shadow *s, struct blkfront_info *info,
			 */
			rq_for_each_segment(bvec, s->request, iter) {
				BUG_ON((bvec->bv_offset + bvec->bv_len) > PAGE_SIZE);
	-                       i = offset >> PAGE_SHIFT;
	+                       if (bvec->bv_offset < offset)
	+                               i++;
				BUG_ON(i >= s->req.u.rw.nr_segments);
				shared_data = kmap_atomic(
					pfn_to_page(s->grants_used[i]->pfn));
	@@ -862,7 +863,7 @@ static void blkif_completion(struct blk_shadow *s, struct blkfront_info *info,
					bvec->bv_len);
				bvec_kunmap_irq(bvec_data, &flags);
				kunmap_atomic(shared_data);
	-                       offset += bvec->bv_len;
	+                       offset = bvec->bv_offset + bvec->bv_len;
			}
		}

打上这个patch, 果然解决parted的问题. 为什么parted会触发这个条件, 而系统正常运行往往
不受干扰?

* 首先这个copy只发生在READ操作
* parted在分配内存的时候(posix_memalign)的时候是sector size (512B)对齐的, 而不是page对齐, 所以bv_offset可以不为0
* parted使用directIO模式, 而page cache模式下的磁盘访问往往是page对齐的
* 连续读大块磁盘, 这里是16KB, 所以在rq_for_each_segment会有多个loop

知道这些以后, 我们可以很轻松地把上面的测试用例改造一下, 就可以reproduce这个问题.

## 问题过后还有问题

不过可惜的是, 这并没有从根本上解决问题, 这里的矛盾在于request_fn (blkif_queue_request)使用的是sg (scatterlist), 而complete函数 (blkif_completion)使用的却是request, 这两个结构的映射并不是简单操作offset的结果, 在blk_rq_map_sg():

	if (sg->length + nbytes > queue_max_segment_size(q))
		goto new_segment;

	if (!BIOVEC_PHYS_MERGEABLE(bvprv, bvec))
		goto new_segment;
	if (!BIOVEC_SEG_BOUNDARY(q, bvprv, bvec))
		goto new_segment;

	sg->length += nbytes;

刚好在我们的环境下, I/O还是会出现corruption. 我们的系统里并不会直接操作xvd设备, 而是在上面再加了一层自己的RAID. 我们的RAID使用4KB的stripe unit size, 但是支持最小512B (sector size)的读写操作. 如果要读取1KB的数据, 而directIO的buffer又不是page aligned, 那么会出现这种情况:

* 0 - A, page P1
* A - (A+1K), page P2, directIO buffer
* (A+1K) - 4K, page P3

因为不能写到directIO之外的地址, 而RAID读取xvd又是以4KB为单位, 所以P1 != P2. 这样在blk_rq_map_sg()里面并不能merge, 但是blkif_completion()却想当然了.

这个问题在upstream也已经解决了, 简单地说就是在requset_fn和completion的时候都使用sg就好了.

## 问题回顾: 没必要的问题?

那么问题又来了, 这个sg真的有用吗? 到最后还不是通过4KB的shared page跟backend交互, 并不像大部分硬件HBA支持更大的segment.
