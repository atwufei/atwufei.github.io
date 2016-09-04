---
layout: post
title: Plugging for 2x RAID sequential throughput
author: JamesW
categories: Storage
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

## 为什么要plugging

[Explicit block device plugging](https://lwn.net/Articles/438256/)中, Jens Axboe提到plugging的主要目的在于:

	The idea behind plugging is to allow a buildup of requests to better utilize the hardware and to allow merging of sequential requests into one single larger request. The latter is an especially big win on most hardware; writing or reading bigger chunks of data at the time usually yields good improvements in bandwidth. 

在后面的评论里, Neil Brown提到:

	Rather than thinking of it as 'plugging' it is probably best to think of it as early-aggregation. hch has suggested that this be even more explicit. i.e. the thread generates a collection of related requests (quite possibly several files full of writes in the write-back case) and submits them all to the device at once. Not only does this clearly give a good opportunity to sort requests - more importantly it means we only take the per-device lock once for a large number of requests. If multiple threads are writing to a device concurrently, this will reduce lock contention making it useful even when the device queue is fairly full (when normal plugging would not apply at all).

现在的explicit plugging机制, 直接在最上层I/O submitter做plug/unplug, 那里也是最了解I/O pattern的地方, 比如文件系统, 内存管理和RAID. 虽然说下面iosched甚至是磁盘内部都有相应的queue可以做merge, 但是上层不只能够做到更好的merge, 而更少的request对于下层操作也大有帮助.

## 怎么使用plugging

plugging的使用主要就是2个函数:

	struct blk_plug plug;

	blk_start_plug(&plug);
	submit_batch_of_io();
	blk_finish_plug(&plug);

注意blk_start_plug甚至不会把I/O下发到request queue里面去, 所以必须要求submit_batch_of_io运行的时间足够短, 不至于让磁盘长时间idle.

## 我们的I/O Stack

在讨论我们的具体问题之前, 先来看看我们的I/O Stack

![I/O stack]({{ "/css/pics/plug/io-stack.png"}})

* raid1使用3份拷贝, stripe unit size 4KB
* raid0使用256KB的stripe size, 也就是stripe unit size 64KB

至于我们为什么使用这种RAID配置, 原因暂且不表. 这虽然比较浪费空间, 但它只使用硬盘的极小部分, 而我们的业务逻辑使用的是另外一种RAID配置.

## 问题定位

最简单的分析就是首先测试以下底下disk的带宽, 然后x4就应该是最上层raid0的带宽, 但是在测试中发现, raid0的带宽远小于这个理想带宽, 那么问题会出现在哪里呢? 最简单的办法就是测试以下每一层的带宽:

1. 单个disk的带宽, 注意这里必须使用组成raid的那个分区, 因为磁盘不同位置的带宽会相差很大.
2. 单个dm (multipath)的带宽, 跟disk带宽基本一致
3. 单个raid1的带宽, 性能损失非常小
4. raid0的带宽, 像之前所说, 远小于disk x4

一般来说, raid0/1因为不需要而外的计算, CPU不可能是瓶颈的. 那么应该就是I/O stack自己的问题了:

* 会不会HBA成了瓶颈? 事实上还真可能, 在这个测试里面, 一个HBA带了12个HDD, 有的HBA虽然理论带宽比这高不少, 不过实际带宽还是上不去.
* raid0能有什么额外开销? 应该不可能, 那么同时测试4个raid1, 照样重现问题, 所以问题跟raid0本身无关.
* 用iostat可以看出, 在顺序读写raid0:

	* raid1处avgrq-sz等于64KB, 看起来就是stripe unit size, 这个对于raid0倒是正常
	* disk处avgrq-sz接近8KB, 这个远远小于期望值64KB, 至于为什么是8KB而不是4KB, 这是因为我们RAID的实现有些特殊处理

依据HDD的物理设计, 大块I/O是很有助于提升磁盘的带宽的. 为什么在这里8KB的request还能达到一定的带宽, 是因为测试的都是顺序I/O, 磁盘enable read cache. 如果通过cache_type disable read cache, 我们可以看到带宽会急剧下降.

## RAID的实现

现在基本上能够判断问题出在RAID实现上面了, 结合代码可以很容易看出RAID在处理大块request的时候, 是把它按照stripe unit size切分后在发送到下一层, 相当程度浪费上面merge的效果.

![raid]({{ "/css/pics/plug/raid.png"}})

## 使用plugging加速

解决这个问题的思路比较简单, 但是注意blk_start_plug/blk_finish_plug之间的时间间隔必须要短, 对于raid来说, 只要做到发送下去的I/O和接收到的I/O一样大就好, 并不用去太多考虑是不是还能再优化一下I/O的大小, 毕竟上层已经通过blk_start_plug/blk_finish_plug尽量优化了.

当然在实现的时候往往会碰到别的问题, 比如我们有一个全局变量限制了in flight数据的总量. 至于怎么发现是这个全局变量的问题, 刚好又是利用plugging的知识, plugging把很多变化的东西转化成不变的, 如果观测到request的大小不是期望的, 就可以一步一步跟踪下去.

## 问题回顾

这本身并不是个小问题, 但在我们目前的系统里, 它并不在关键路径上, 所以这类问题往往会被人忽视. 不过事情往往是变化的, 当我们去丰富产品features的时候, 这种raid配置出现在关键路径上也不会奇怪.
