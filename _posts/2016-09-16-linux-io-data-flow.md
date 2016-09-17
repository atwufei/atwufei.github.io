---
layout: post
title: Linux I/O data flow
author: JamesW
categories: Storage Kernel
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

本文不讨论复杂的I/O栈, 只考虑最简单的一块硬盘的情况, 没有了multipath, raid等, 反而更有助于理清I/O的本质.
I/O栈主要处理以下这些问题, 它们也决定了这个过程中需要使用的数据结构以及处理方法.

* 数据从哪里来
* 数据到哪里去
* 怎么去的

## 数据表示

数据的来去决定了使用的数据结构.

### bio

一般来说, I/O的数据主要就是文件系统的page cache和metadata, raw device的page cache, 或者是directIO中userspace提供的buffer, 对于这些系统:

* 它们的管理单元是page, 或者更小的单位block
* 它们并不直接跟底层的硬件打交道, 所以也不需要它们的具体细节

针对这种情况, 内核提供了struct bio来描述读写的数据, bio主要包括:

* bio_vec的数组, 每个bio_vec都对应一个page, 以及offset和len. 这并不说明bio_vecs在内存上就不连续.
* 读写的设备struct block_device, 已经设备的偏移bi_sector. 这也说明同一个bio的数据在磁盘上是连续的.

这是Linux Kernel Development里面的一幅图:

![bio]({{ "/css/pics/io/bio.png"}})

### request

虽然struct bio只是上层用来描述数据的, 但是它已经包含I/O的源和目的, 所以在块设备驱动里直接使用bio并无不可, 比如一些NVRAM驱动就是这么使用的(使用blk_queue_make_request注册自己的make_request_fn函数). 更普遍的情况, struct bio在blk_queue_bio()里会转化为struct request:

* request在磁盘空间是连续的
* request是后面merge, sort, dispatch的对象
* request的大小受request_queue.queue_limits限制, 主要包括:
	* max_segments, segment物理上是内存连续的
	* max_segment_size, 如果支持cluster属性, 一个segment可以大于PAGE_SIZE
	* max_sectors_kb
* 一个request包含一个或多个bio 
* 一个bio如果太大, 可能会被split成几个request

### scatterlist

驱动程序在拿到request之后, 往往需要转化成scatterlist. scatterlist只描述数据:

	struct scatterlist {
		unsigned long   page_link;
		unsigned int    offset;
		unsigned int    length;
		dma_addr_t  dma_address;
	};

scatterlist目的有2个：

* 因为bio_vec最多只能描述一个page大小, scatterlist把物理内存地址连续的相邻bio_vec放到一个segment. 也就是说scatterlist里面的length不再局限于一个page, 它的最大值可以达到max_segment_size.
* 在使用iommu的系统中, 可以把物理地址不连续的内存segment映射到设备可见的连续地址空间.

### 小结

* bio: I/O层的用户, 比如文件系统使用block_read_full_page/mpage_readpage把连续磁盘物理块的访问转化为bio, 这也说明, 如果一个page的blocks在磁盘是不连续的, 那么会生成多个bio
* request: 把磁盘物理块相连的bio merge在一起, request是iosched的主体
* scatterlist: 既保证物理块连续, 也保证内存上连续, 是DMA的对象. 如果存在IOMMU, 可以进一步优化.

## I/O Flow

I/O子系统提供只给上层提供了submit_bio一个函数, 而submit_bio的主要任务就是调用q->make_request_fn(), 而我们上面已经讲过, 只有像NVRAM这样比较特殊的设备才会自己定义, 绝大部分的物理设备驱动都是使用同一函数blk_queue_bio.

### blk_queue_bio

#### 正常流程

blk_queue_bio的目的很简单, 就是把bio挂入到相应的queue里面, 这样设备驱动就可以拿去处理了. 之前讲过, request才是iosched/driver的对象, 所以这个函数需要把bio和request联系起来, 有2种方法:

* 最简单的就是直接挂到一个新生成的request上
* 或者可以merge到一个已有的request上

#### congestion

如果I/O来的太快导致下面处理不过来, 就需要设置相应的congestion标识通知上层暂时不要再发I/O, 而当前的进程则进行等待, 这都是在get_request里面做的.

TODO: 增加更多细节.

### queues

一个I/O的生命周期会经过不同的queues.

#### current plug list

关于[Explicit block device plugging](https://lwn.net/Articles/438256/)的作用, 这里不再重复, 有兴趣也可以参考[Plugging for 2x RAID sequential throughput](http://wufei.org/2016/09/04/plug-for-raid/).

每个进程都维护自己的一个plug list, 这也意味着这个list的操作, 包括merge, 不需要上锁, 同时也说明这个list上的requests是驱动不可见的, 只有等到unplug时才会把requests加到request_queue去, 当然unplug不一定需要显式调用blk_finish_plug.

#### request_queue

request queue是这里的核心, I/O用户或者plugging把requests放到request queue上等待I/O, 下层驱动从request queue取出request去做I/O. 对于这两种操作, request_queue分别提供了两种不同的queue:

* 每种不同的iosched通过q->elevator->elevator_data维护一条queue, 当然具体实现可以是任何数据结构, 用来接收上层的requests.
* request_queue通过q->queue_head维护一个dispatch queue, 底层driver按顺序service. iosched按照自己的优先级逻辑把request从上面那条queue的request移到dispatch queue里面.

就像上面说的, request_queue中的request的个数超过一定限度, I/O系统就处于congestion状态, 这个是由q->nr_requests决定的. 但注意这并不是hard limit, 具体实现可以参考get_request.

### iosched

iosched 主要做两件事情merge和sort.

#### merge

* merge的目的. 因为HDD, 甚至SSD都prefer大块I/O, merge对盘的性能有很大影响. 当然像NVRAM之类的是个例外, 所以它们一般不用进行merge.

* merge的时机. 最好的莫过于在plug list上了. 因为是同一线程的I/O, merge的可能性更大, 而且这个过程并不需要hold住q->queue_lock. 由于前后两个I/O merge的可能性更高, 所以plug list是反向遍历的. 即使这样, list的遍历还是慢的, 所以对plug list长度有一定的限制, 也就是说某种情况下是会隐式unplug的. 当然不同进程之间的I/O也是可以merge的, 这个merge操作就只能在加入request_queue上做.

* merge的类别. 新加入的bio可以加到一个request的后面back merge, 也可以加入到request的前面front merge. 一般来说, 文件系统的I/O back merge的可能性更高, 而front merge则不一定有效, 所以在deadline iosched里面front merge是可以disable的, 这是因为attempt merge这个操作也是需要一定的时间, 并且还需要lock住q->queue_lock. 而在plug list的merge会同时考虑back/front merge, 因为开销并不大. 当然当新加入的I/O能够把前后两个requests连接起来是, request之间也是可以merge的. 

* merge的花销. 针对back merge, iosched的通用层实现了一个hash表来快速定位合适的request. 对于front merge, 这个操作则让每个iosched自己来选择, 比如deadline就通过自己的rbtree来查找, 可见会有一定的开销.

* merge的标准. 并不是说bio跟某个request的LBA地址相连就可以merge, 这只是其中的一个必要条件, 还主要取决于queue_max_sectors(q)和queue_max_segments(q), 还有一些其他的约束条件. btw, 如果不是文件系统的request, 使用的是queue_max_hw_sectors(q)而不是queue_max_sectors(q), 具体可以参考ll_back_merge_fn().

#### sort

sort的目的主要有两个:

* 对于HDD, 为了减少磁盘seek/rotate的时间, 把request按照磁盘的顺序进行排列能够有效减少这部分开销. 当然对于SSD, 这样做并没有太大影响.

* 对于不同的iosched, 会有自己的优先级考虑. 比如说deadline是不是到了, 是不是REQ_SYNC, 是不是要考虑fair, 等等不一而足. 所以iosched需要对request进行相应的排序, 然后在dispatch的时候按照这个顺序提供给底层driver (elv_dispatch_sort会插到dispatch的中间, driver还是从头取).

### driver

scsi_request_fn()

### io barrier

为了实现I/O的依赖关系, 需要实现io barrier以消除这个过程中产生的乱序情况, 比如journal里面metadata需要先完成, 然后commit record才能够完成.

I/O栈乱序有两个原因:

* iosched允许乱序
* disk里面有write cache, 写返回的时候并不意味着data已经写入盘面

以前解决这个问题的办法, 也就是实现hard barrier的方法:

* 针对iosched的乱序, 使用drain request queue的办法, 保证前面的request已经完成
* 针对write cache的办法, 使用FLUSH命令, 或者disable cache都可以解决, 也可以结合FUA使用

新的解决办法是让应用层也就是文件系统来解决这个依赖关系, 而不是drain queue, 当然write cache该怎么办还得怎么办. 但是这个会带来明显的好处吗? 注意FLUSH会去清空write cache, 其他的write request在这个过程中还能进行吗? LWN的Corbet在[The end of block barriers](http://lwn.net/Articles/400541/)的评论说:

``The cache flush can happen in parallel with other I/O. Forcing specific blocks to persistent media can only slow things down, but they have to get there soon in any case. While the drive is executing the cache flush, it can be satisfying other requests whenever it's convenient. "Cache flush" doesn't mean "write only blocks in the cache" or "don't satisfy outstanding reads while you're at it". It will be far more efficient than a full queue drain.``

不过原来的那种drain queue方式难道不能只drain write request吗?

## iostat

### 实现细节

### queue theory

## blktrace
