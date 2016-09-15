---
layout: post
title: Linux Filesystem Primer
author: JamesW
categories: Kernel Storage
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

# 什么是文件系统

文件系统简单地说就是完成两个任务:

* 怎么找到文件, 也就是文件之间的组织方式
* 怎么访问文件内容, 也就是文件内部的组织方式

我们一般比较倾向于使用树形结构来定位一个事物, 比如我老家是江西省XX市XX县的, 这样表达的好处就是简单并且容易沟通, 如果是老乡的话就更加有共鸣了. 大部分文件系统也是一样, 文件通过目录组织起来, 目录里又可以再嵌套子目录, 只要每层目录都有具体的含义, 很容易通过遍历目录来找到具体的文件. 当然凡事也都有例外, 不同场合可能会有不同选择, 有的时候我们可能就是需要扁平的结构, 或者说所有的文件都存放在根目录下, 比如到了美国我们就都是Chinese了, 具体哪个省对美国人来说也没有什么意义. 而且由于省去了中间目录, 反而可以更快地定位文件, 有的分布式文件系统通过简单的Hash算法就可以决定文件和节点之间的映射. 对于具体的EXT2文件系统, 文件是可以通过路径来定位的, 路径包含了经过的所有目录和最终的文件名.

有了目录, 文件, 和文件内容, 我们应该怎么组织呢? 至少需要做到两点:

1. 通过目录, 可以找到该目录下的所有文件以及子目录
2. 找到文件之后, 能够访问该文件的所有数据

我们需要为它们分别建立一张表来记录这种对应关系, 但是怎么通过*文件名*把这两张表联系起来呢? 毕竟我们的根本目的是通过文件路径访问文件的数据. 一个简单的办法就是把表2直接集成到表1里面去.

![]({{ "/css/pics/fs/dir-file-1.png"}})

虽然能够完成任务, 但却不够好. 表2的条目相对较大, 需要存储数据块的引用, 还包括一些其他的元数据, 这样就会导致一条目录项变得很大, 从而导致一个block能够只能存储有限数目的文件, 这样对文件的查找是非常不利的, 因为可能需要更多的磁盘访问. 但是如果直接使用文件名去关联2张表的话, 也不是什么好主意, 所以在Unix文件系统里, 每一个文件都会对应到一个数字(index), 通过这个文件系统唯一的编号就可以快速地找出文件所有内容的磁盘位置. 简单地说, inode就是这个index, 再加上其他的一些元数据. inode还有另外的一个小小的好处, 就是可以很容易实现硬链接(hard link).

![]({{ "/css/pics/fs/dir-file-2.png"}})

至于文件内部的组织方式, 不同的文件系统更是千差万别, 可以是树形的, 也可以是简单索引, 甚至还可以让别的文件系统来解析. 一般而言, 文件系统的最小单位是数据块(block), 而一个block往往会包含一个或者多个磁盘的扇区(sector), 扇区是磁盘访问的最小单元.

# EXT2 文件系统

EXT2作为经典的文件系统, 又不失简单高效, 非常适合入门.

## 磁盘分布

想要深入理解EXT2, 了解它的磁盘分布(Disk Layout)是必不可少的. 首先来看看它的整体分布.

**(本文大量使用 Understanding the Linux Kernel 的图表)**

![]({{ "/css/pics/fs/ext2-layout.png"}})

首先整个磁盘或者分区分成一个个的block group, 为了防止文件系统碎片, EXT2会尽量把一个文件分配到同一个Group里面, 保证更高效的访问文件. 从上图可知, 每个block group中只有一个block用作data block bitmap, 每个group的最多能存放 8 * block size * block size字节, 如果block size是4KB的话, 那么一个group可以存放128MB数据.

一个block group包含这些元素:

* super block: 用于记录整个文件系统的信息. 虽然每个group都包含了一个super block, 但是 并不要求严格统一. 话说即使这些super block都是同步的, 如果group 0 的super block坏了, 是否就可以通过其他的来修复呢? 我们当然是这么期望的, 不过现在好多的disk, 不要说AFA, 即使是普通的SSD盘, 都有可能集成去重功能(Dedup), 也就是说相同的数据在磁盘上可能只有一份拷贝.  也就是说, 上层文件系统想通过冗余来改善系统的稳定性, 下层却把冗余给去重了. 这样看来, 同步倒还不如不同步了, 不同步至少减少了Dedup的可能. 由此可见, 一个好的约定, 理解对方的需求是多么重要, 否则两边都给出了完美的解决方案, 合在一起却不能达到预期的效果.

* group descriptors: 每个group的相应信息, 主要包括data block, inode bitmap和inode table的位置信息, 已经空闲block和inode的个数. 每个group不止包含自己的descriptor, 还有可能包含其他group的descriptor, 也就是说有冗余. 虽然说这个descriptor的数据结构并不大, 但是每一个descriptor都占用了一个block, (如果多个group descriptors共用一个block, 担心修改其中的一个group descriptor会影响到其他的?  不太应该). 对于一个很大的文件系统, 如果每个group都需要保存所有的group descriptors, 这个space overhead会非常大, 所以应该是可配置的, 见descriptor_loc().

* data block bitmap:
* inode bitmap: 使用bitmap来标记对应的block或者inode是否已经分配, bitmap很适合快速地查找和分配空闲空间.

* inode table: inode table顾名思义就是存放inode的地方, 每个inode的大小是一样的, 所以根据inode的号可以很容易就找到对应的inode. inode里面记录了文件相关的元数据, 比如说Owner, Mode, Size, 和各种各样的time, 更重要的是包括了文件block的指针, 通过这些指针就可以找到某个具体offset对应的内容. 中间的某些block指针为0的情况也是允许的, 甚至变成了一个feature, 也就是hole, 这种情况下文件的size和磁盘的使用是不相等的.

那么inode到底是怎么来描述一个文件的呢? 很多文件系统的教科书上都会有介绍, 而且基本上是跟EXT2一致的. 这种数据结构一定程度上体现了简单有效, 如果是小文件的, 可以很方便地使用12个direct addressing的指针就可以表示出整个文件, 如果更大的文件, 可以通过indirect block来引用. 虽然谈不上完美, 比如每个引用只能索引一个block, 也能较好的解决了问题.

![]({{ "/css/pics/fs/inode.png"}})

EXT2的元数据也就以上几种, 不过还有一种数据比较特殊. 由于文件名的长度各不相同, 为了节省空间, EXT2使用如下变长的结构, 在结构里面通过rec_len来表示该结构的长度. 如果目录是一直增加的, 那实现起来非常简单, 不过怎么支持文件删除, 删除了的空间怎么再次得到利用, 一定程度上增加了目录操作的复杂度, 有兴趣的可以考虑一下下图中inode 67的rec_len. 这个目录结构虽然紧凑, 不过要在其中找一个文件, 却没有什么好办法, 只能从头到尾扫描, 如果目录里面有很多文件或者子目录, 扫描的效率不会太高. 如果目录项根据文件名排好序的话, 检索特定的文件就会非常方便, 很多更现代的文件系统中使用btree来达到这个目的.

![]({{ "/css/pics/fs/dir.png"}})

## 性能优化

了解了以上所说的disk layout, 对EXT2可以说已经有个初步理解. 对于文件系统设计来说, 还有一个很重要的问题就是怎么保证文件访问的性能, 特别是怎么优化磁盘I/O, 虽然上面或多或少已经有所考虑, 但是还有2个方面没有涉及, 特别是data在哪里分配决定了文件系统的性能, 毕竟大量的文件操作是访问data, 而不应该是metadata.

### inode在哪里分配

如果是普通文件, 使用find_group_other()得到新创建文件inode的block group.

1. 尽量和父目录放在一起. 如果只考虑文件本身和它的父目录, 这个会有意义吗? 这样的locality会带来读时的优化, 还是写时的优化? 其实并没有. 一般来说, data会和inode同属一个bg (block group), 读文件的时候, 先去读取父目录的data, 这里面会记录文件的inode, 然后从inode里面取得block的信息, 注意inode在bg里是放在开头的, 也就是说读的时候并不能减少seek时间, 甚至相反. 在创建的时候, 因为要同时修改文件的inode和父目录的inode, locality会带来好处? 不过和其他的rule联合起来, 这样做却很容易把有访问locality的文件聚集到一起, 从而提高性能. 如果真有必要弄清楚, 通过trace可以很容易作出判断, 有时候想再多还不如动一下手.

2. 尽量把相同目录下的文件放在一起. 假设用户访问文件有locality, 比如多进程编译kernel的时候, 同一个目录的文件会集中访问, 放在一起就可以有效地减少seek的时间 (不只inode, 还有inode对应文件的data block). 注意底下的iosched能对I/O进行排序和合并.

3. 如果父目录所在的bg已经满了, 尽量岔开不同的目录. 如果还把这些目录的文件放在一起, 根据之前的经验, 新的bg也会有更大的概率分配完. 这其实和2是一致的. 对应的代码如下:

		group = (group + parent->i_ino) % ngroups;
		for (i = 1; i < ngroups; i <<= 1) { /* select one from these groups */}

对于新创建的目录, 老的方法find_group_dir()并没有讲究太多, 只是选出一个free inodes和free blocks相对较多的block group. 新方法find_group_orlov()考虑的更多一些, 不再详述. 如果有兴趣, 可以阅读一下这篇文章 [Locality and The Fast File System](http://pages.cs.wisc.edu/~remzi/OSTEP/file-ffs.pdf)

### data在哪里分配

data block的分配可能显得更加重要, 访问data block一般来说比metadata多得多, 访问时locality更强, 碎片导致的性能问题也会更大. data block, 包括indirect block, 都在ext2_get_block()里分配, 更具体点就是ext2_find_near():

1. 尽量靠近前面的block
2. 尽量靠近上一层indirect block
3. 尽量放在inode的bg

除了这些努力之外, 另一个有效的办法就是预先reserve一片连续的磁盘空间, 虽然get_block()会尽量就近分配, 但有时却无奈其他文件分配的干扰. 比如文件A已经分配了block n, 本来正打算分配下一个block, 最好的选择就是block n+1, 可是文件B恰好就已经分配了block n+1. 为了避免这种情况, 预先reserve [n, n+window]是个比较好的选择. 更妙的是, 这个reserve并不需要浪费物理磁盘的空间, reserve整个是在内存中完成的, 当没有人再打开该文件的时候, 就可以释放reserve的区间. 一般来说, 我们会一次增加大量的数据, 而不是每打开一次增加一两个数据块, 所以这种内存reserve方式可以很好的工作.

所有的这些尝试主要目的就是减少磁盘寻道的时间, 要检验它们的效果, 可以试试blktrace, seekwatcher等.

# 文件的访问

## page cache

为了加速文件的访问, 系统中往往会使用某种类型的cache, 除非指定directIO, Linux中文件的读写都会经过page cache. 和所有的cache一样, page cache的好处显而易见, 可以很大程度提高读写的速度, 不过又因为Cache的存在, 数据并没有直接写到disk上, 所以有数据丢失的风险. 在更早以前, 同一个block甚至会在内存中存在于2个子系统之中, 分别为page cache和buffer cache, 当然现在已经合二为一, 对这段历史感兴趣的可以参考 [UBC: An Efficient Unified I/O and Memory Caching Subsystem for NetBSD](https://www.usenix.org/legacy/event/usenix2000/freenix/full_papers/silvers/silvers_html/)

即使是现在, /proc/meminfo中还是有:

	Buffers:           96468 kB
	Cached:          1698812 kB

不过Buffers的值表示属于block device的page cache, 比如直接去访问raw device, 或者是文件系统的metadata. Cached的值表示普通文件的page cache.

	bufferram = nr_blockdev_pages();
	cached = global_page_state(NR_FILE_PAGES) - total_swapcache_pages – i.bufferram;

虽说现在已经没有单独的page cache和buffer cache, 但是同一磁盘block还是可能存在于不同的内存page中.

* 通过普通文件接口访问一个block
* 通过block device访问同一个block

kernel并不会保证这两份内存数据保持同步, 这是使用者需要考虑的事, 一般来说并不推荐同时使用这2种方法同时访问一个block. 直接使用dd之类操作block device当然可以避免, 但访问文件系统的metadata却不能避免, EXT3的时候会讲到, 这就要求EXT3的代码小心处理.

### address_space

那么page cache是怎么组织的呢? 属于同一文件, 更精确地说, 同一inode的page cache会集中起来放到一个address_space的数据结构中, 该数据结构使用radix tree来存放page cache. radix tree并不复杂, 主要就是实现文件偏移到page的映射, 所以radix tree有点类似页表, 只不过页表是固定层级. address_space另一个重要的成员是address_space_operations, 这些回调函数定义了怎么操作page cache, 比如要读入一个page应该怎么做, 每个文件系统都需要实现这个接口. 

![]({{ "/css/pics/fs/radix-tree.png"}})

### buffer_head

虽然已经没有单独的buffer cache, 文件系统的最小单元依然还是block, 一个page可以包含一个或多个block. 在文件系统创建的时候, block size就确定了. 当用户通过read/write系统调用访问某块disk block的时候, 一般会经过这些步骤:

1. 映射[struct file, file->f_pos]到一个page, 该page->mapping对应到这个文件, page->index对应到f_pos
2. 映射page到若干个buffer_head, 这里主要的任务是根据[page->mapping, page->index]找到block在块设备上的地址, 也就是[bh->b_bdev, bh->b_blocknr], 这需要文件系统get_block的帮助. 有了buffer_head的这些信息, 就可以通过submit_bh去读写磁盘上的数据.

简单地说, buffer_head就是用来记录page到磁盘block的映射.

![]({{ "/css/pics/fs/buffer_head.png"}})

## file access

### read

我们已经说过, 只要不是directIO, 所有的读写操作都会经过page cache, 所以读写操作都是围绕着address_space进行的.

1. 首先在page cache里面查找, 如果存在并且uptodate, cache命中, 直接拷贝到用户空间的buffer就好. 一般来说, Linux系统倾向于尽可能多的cache page, 真到需要free page的时候, 回收clean page cache是很快的.
2. 如果不在page cache, 或者不是uptodate, 那就需要从磁盘里面去读取数据, 并存放到page cache. 这个过程会使用到address_space->a_ops->readpage(), 也就是说每个文件系统都需要实现一个address_space_operations, 由它来实现page cache的读取和写回. 对于大部分文件系统, 比如EXT2, readpage的实现会使用通用的函数加上自己的get_block()函数, 毕竟只有文件系统自己知道自己的disk layout, 也就是page和block number的mapping.

这里说的通用函数有2个选择:

* mpage_readpage
* block_read_full_page

这2个函数的主要区别就是要不要使用buffer_head

* 如果该page对应的block刚好在disk上是连续的, 那么完全可以使用一个bio下去
* 如果blocks不连续的话, 那mpage_readpage也没有办法, 同样调用block_read_full_page

block_read_full_page为page里面的每一个block都生成一个buffer_head, 然后使用submit_bh把block一个一个往下发. 这2种方法并没有什么本质区别, 按理说mpage_readpage会稍微好点, 毕竟能够省掉buffer_head的空间开销, 但是对于disk I/O本身区别并不大, 即使拆分成多个bio下发, iosched仍然会尽可能的merge I/O, 所以文件系统对调用哪个函数并不特别讲究. 即使EXT2在readpage的时候使用的是mpage_readpage, 只要有write这个page的操作, 也一定会生成buffer_head, 更可以看出这两种方法并无太大区别.

### write

相对于read, write复杂很多. 我们先来看一下generic_perform_write, 它的主要逻辑如下:

	do {
		status = a_ops->write_begin(file, mapping, pos, bytes, flags, &page, &fsdata);
		copied = iov_iter_copy_from_user_atomic(page, i, offset, bytes);
		status = a_ops->write_end(file, mapping, pos, bytes, copied, page, fsdata);
		iov_iter_advance(i, copied);
	} while (iov_iter_count(i));

write复杂的逻辑主要在write_begin, 对于EXT2它会做这些事情:

* 找到或者创建page cache, bufferred write必须经过page cache
* 找到或者创建page cache对应的buffer_head
* 调用get_block()读入整个chain, 从ext2_inode.i_block一直到指定的data block, 如果要写入的地址还未分配, 比如写到文件末尾或者是空洞, get_block会在磁盘上分配相应的空间
* 如果get_block新分配了空间(buffer_new), 需要调用unmap_underlying_metadata(bh->b_bdev, bh->b_blocknr), 该调用会清除buffer cache alias (blkdev). 至于为什么要调用unmap_underlying_metadata(), 后面有讲.
* 如果只是部分写block, 那么需要把磁盘上的block先读进来

### mmap

除了read/write系统调用外, mmap也可以用来访问文件. mmap把page cache直接映射到用户态地址空间, 在访问的时候便不再需要用户空间和内核空间之间拷贝, 所以对性能有一定的好处. 不过混用read/write和mmap并不是一个好的编程实践, 虽然Linux会尽量保证其正确性, 有的OS并不保证其正确性, 也就是说要么使用read/write, 要么使用mmap, 不要同时使用.

同时使用read/write和mmap的结果就是同一个物理page可能会产生两个虚拟地址, 一个是mmap的用户态地址, 另一个是read/write时在内核[临时]映射的地址, 有alias就有可能有麻烦(就像page cache和buffer cache一样), 它们可能映射到不同的cache line里面, 也就导致没有机制对它们进行同步. 这种情况在VIVT和VIPT cache的CPU上有可能发生, 考虑这种情况:

        int val = 0x11111111;
        fd = open("abc", O_RDWR);
        addr = mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
        *(addr+0) = 0x44444444;
        tmp = *(addr+0);
        *(addr+1) = 0x77777777;
        write(fd, &val, sizeof(int));
        close(fd);

在我的这个[patch](https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=931e80e4b3263db75c8e34f078d22f11bbabd3a3) (btw, 我为什么要故意用这个名字) 没有checkin之前, 文件的前两个integer变量有的时候会是0x44444444, 0x77777777, 并不总是我们期待的0x11111111, 0x77777777.

# EXT3 文件系统

EXT2是个相对高效的文件系统, 它最大的缺陷在于没有足够的完整性保障. 在系统崩溃或者掉电的情况, 文件系统会处于一个不一致状态, 从而需要通过fsck去扫描整个文件系统, 并期待它把文件系统的metadata恢复到一致状态, 如果文件系统很大, 这是一个相当耗时的过程.  EXT3相对EXT2的主要改进就是通过journal来解决问题, 一般情况下EXT3的性能比EXT2还略差, 但是提高了完整性, 特别是metadata的完整性.

## 日志

### 事务机制

#### update-in-place

我们都知道, 所有的不一致性都来源于update-in-place. 如果直接覆写原有的地方, 数据就可能处于两个稳定状态之间的某个临时状态, 这个时候系统crash, 文件系统便会处于不一致状态. 解决的办法只有一个, 就是不要使用update-in-place, 当然怎么做到这点还是有不同方法的:

* Log-structured fs - 使用这种方式的文件系统把新数据都写到空闲数据块, log可以是append模式, 也可以是系统中任意空闲的地方. 因为系统中存放了不同版本的数据, 所以可以很容易实现snapshot功能. 
* Journaling fs - 把新数据写到一个专门的地方, 可以是一个文件系统内的一个特殊文件, 或者一个单独的设备. 这种文件系统最终还是会overwrite文件系统, 不过是在日志存在的情况下做的, 也就是说文件可以是修改前的状态, 或者修改后的状态, 但是两者只能选其一, 而不像Log-structured fs可以同时访问不同的版本.

日志的实现也有不同的办法, 有的在日志里面存放更新的操作, 这种方法能够节省日志的空间, 有的在日志里存放更改后的数据, 这种方法更简单一点. EXT3的日志属于后一种.

#### 原子操作

那么什么操作物理上是原子的? 从磁盘角度看, 我们可以简单认为一个sector的读写是原子的, 要么一个字节都没写, 要么全写. 一般来说, 这个假设是没有问题的, 不过像数据库可能会有更宽松的假设 (假设的越少越可靠), 比如SQLite并不要求sector的写操作是原子的, 但是如果第二个字节被更新了, 可以认为第一个字节一定也被更新了. 有了这个基本的原子操作, 就可以很容易构建逻辑上的原子操作, 比如要写多个磁盘sector, 那么当且仅当最后一个commit sector被写入了, 才认为这整个操作完成.

#### I/O顺序

并不是说最后下发的I/O就会最后完成, 普通情况下I/O是乱序执行的, iosched可能会打乱顺序, 有的时候为了性能甚至是必须的, 磁盘本身也不保证顺序执行. 但是日志的commit操作肯定是有I/O顺序要求的, 以前的kernel是通过block层的barrier实现的, 基本思想就是drain queue然后flush disk write cache, 这样肯定会导致磁盘的性能大大降低, drain queue之后就不能接收I/O了. 之后的实现方法是让上层自己来实现, 而block layer只要实现flush操作, 也就是把数据flush到物理介质上. 比如jbd要实现transaction操作, 它只需要:

1. 下发transaction的相关数据的I/O
2. 等待1的I/O返回, 注意这个时候并不保证数据一定存放到物理介质上, 有可能还在write cache
3. flush, 并且等待其返回
4. 发送commit I/O
5. flush, 并且等待其返回

4和5可以通过FUA (Force Unit Access)简化. 经过这个优化, block层免去了drain queue的操作, 从而可以保证其他并行的I/O得以继续, 同时磁盘有一定的调整空间, 比如优先flush还是优先别的操作. [The end of block barriers](https://lwn.net/Articles/400541/)介绍了相应的背景.

#### 独立性

事务当然需要保证独立性, 2个事务互相影响那就不叫事务了. 当一个block已经被一个事务修改过, 这个时候新的事务要修改它该怎么办? 无他, 也就只能再拷贝一份. 如果是通过write()系统调用实现的话, 这个很容易实现, 因为文件系统有机会截获这个操作, 从而增加一份拷贝. 可是如果是通过mmap()然后直接写对应的地址空间呢? 文件系统不能感知到这个操作, 也不太可能通过pagefault来截获这个操作, 否则性能受不了. 首先mmap会影响磁盘分配吗? 如果日志模式采用journal怎么办? 这种情况就根本不应该用mmap?

### 日志模式

EXT3实现了3种不同的日志模式, 保护级别由低到高:

* writeback - 只有metadata做日志, data允许 *在metadata写入日志之前* 写入磁盘
* ordered - 只有metadata做日志, data必须 *在metadata写入日志之后* 写入磁盘
* journal - data同样需要写入日志, 由于开销过大, 一般很少使用

那么writeback和ordered的区别在哪?

* 首先这两种模式都能保证metadata的一致性, 因为它们都会对metadata做日志.
* writeback因为不保证data和metadata的顺序, 如果metadata先写入日志了, 但是data block还没有写入, 这个时候crash的话就会看到之前的data block, 如果是多用户系统, 这会是个安全问题. (新分配的data block)
* 如果都是写老的数据块, 那这两种方式没有本质区别, metadata的修改可能只是涉及到一些timestamp的更新而已, 并不会涉及到metadata和data block的映射. 这个时候metadata等待data只是徒劳, 却引入依赖关系.
* 两种方法都没法保证data的完整性, 这个需要上层应用比如SQLite来处理.

虽然从这个角度看, ordered相对于writeback是有那么一点点优势, 不过由于需要保证data/metadata的写入顺序却可能引入性能方面的问题. 需要注意的是, 一个原子metadata的操作不仅仅依赖(等待写完)于它相关的data block, 还有它之前所有的metadata依赖的data block. 在我们公司的系统里, 很多因为fsync导致timeout的问题.

![]({{ "/css/pics/fs/ext3-ordered.jpg"}})

上图中如果要fsync(f6)的话, 需要按序写入所有的data和metadata. 我们也可以用fio来简单测试一下:

	(1) fio --name=write-blocker --filename=data_blocker --rw=write --size=32G --bs=4k --runtime=150 &
	(2) fio --name=write-$type --filename=data --rw=write --size=1G --bs=4k --fsync=1 --runtime=120	

fsync在(2)的延时因为日志模式有很大变化.

![]({{ "/css/pics/fs/ordered-writeback.png"}})

[Solving the ext3 latency problem](https://lwn.net/Articles/328363/)也有相关的讨论.

## 实现细节

### 基本概念

* handle: 表示一个原子操作, 文件系统比如EXT3需要考虑的是把哪些操作加入到同一个handle, 才能保持文件系统的一致性.
* transaction: 表示一个事务. 原子操作是通过事务来实现的, 一个原子操作一般只会修改几个block, 但是对于磁盘来说, 小块的I/O对性能有很大的影响的, 所以JBD把多个handle组合成一个transaction. 这样做还有别的好处, 比如属于同一个transaction的两个handle都写同一个block, 那么并不需要写两次磁盘, 也就是说这个写操作是可以合并的, 当然不能跨越transaction. transaction的事务性通过最后一个commit block (sector) 来实现, 也就是说这个sector写成了, transaction就可以认为已经完成, 如果这个sector没写, 则transaction需要保证它里面的所有handle都没发生. 一个sector写操作的原子性是由磁盘保证的.

### 基本操作

* journal_start: 开始一个原子操作
* journal_stop: 结束一个原子操作

* journal_get_create_access: 取得新分配块的写权限, 对于data != journal的模式, 所谓新分配块就只有文件的indirect block一种情况. 该函数会把输入的buffer_head纳入到jbd管理. 

* journal_get_write_access: 在准备修改某块(和buffer_head有时候混用)之前, 需要调用该函数. 有一点需要注意的是, 如果这个block也出现在之前的transaction(*j_committing_transaction*)里面, 显然两个transaction不同同时修改同一个block, 否则根本就保证不了独立性, 那么在这种情况下就必须copy一份(*jh->b_frozne_data*)供之前的transaction使用. 如果之前的transaction正在往磁盘写这个block的话, 因为不能再copy一份给它了, 只有先等它结束, 否则新的write可能就导致之前的transaction写入不一致的数据.

		block n in t1			block n in t2
		-------------------		-------------------
		|   frozen data   |		| ready to change |
		-------------------		-------------------

* journal_get_undo_access: 释放一个block的时候, 会通过data block bitmap相应的某位置为0. 如果该操作还未写入日志, 就把该block重新分配出去, 那么对应的data block就可能写入了新的数据, 注意data block一般情况(data != journal)下, 是可以在metadata的日志写磁盘之前先写到磁盘的. 这个操作是不可逆的, 因为data block并没有做journal. 所以在释放data block的之前, 需要先调用journal_get_undo_access把data block bitmap拷贝到jh->b_committed_data, 在block分配的时候就会去查这里的内容, 而不是本身的bitmap_bh. 不过为什么在分配的时候(ext3_try_to_allocate_with_rsv)也会调用这个函数?

				   data block bitmap	[block 0 data]
		handle 1: (*0* 1 1 1 1 1 1 1)	[stale data]	<- fail to commit
		handle 2: (*1* 1 1 1 1 1 1 1)	[newle data]	<- data (!journaled) committed before t1

* journal_dirty_data: 在ordered模式中, 如果修改了data block, 需要调用这个函数, 主要作用就是保证data block在metadata之前先commit, writeback模式并不需要, 对于journal模式, 所有的写都是metadata.

* journal_dirty_metadata: 修改了metadata, 调用该函数, 把bh置脏jbddirty. 要想自己管理data/metadata的写盘顺序, 首先就需要避开内存子系统的干预, 它在内存回收的时候并没有文件系统的相关信息, 只要page是dirty, 就可能按照任意顺序进行写盘. 所以jbd并不会像EXT2那样通过mark_buffer_dirty把metadata对应的page设置成dirty, 而只是把bh设置为jbddirty. 这样对于内存子系统来说, 该page是clean状态, 所以并不会触发写盘操作.

data=journal + mmap会有怎样的结果? 一定不太好.

### 事务在EXT3的使用

#### 创建文件

	ext3_create
		handle = ext3_journal_start(dir, EXT3_DATA_TRANS_BLOCKS(dir->i_sb) +
						   EXT3_INDEX_EXTRA_TRANS_BLOCKS + 3 +
						   EXT3_MAXQUOTAS_INIT_BLOCKS(dir->i_sb));

		inode = ext3_new_inode (handle, dir, &dentry->d_name, mode);
			/* 修改inode bitmap */
			bitmap_bh = read_inode_bitmap(sb, group);
			ext3_journal_get_write_access(handle, bitmap_bh);
			ext3_set_bit_atomic(sb_bgl_lock(sbi, group), ino, bitmap_bh->b_data);
			ext3_journal_dirty_metadata(handle, bitmap_bh);

			/* 修改group desc */
			ext3_get_group_desc(sb, group, &bh2);
			ext3_journal_get_write_access(handle, bh2);
			le16_add_cpu(&gdp->bg_free_inodes_count, -1);
			ext3_journal_dirty_metadata(handle, bh2);

			ext3_mark_inode_dirty(handle, inode);
				ext3_reserve_inode_write(handle, inode, &iloc);
					ext3_get_inode_loc(inode, iloc);
					ext3_journal_get_write_access(handle, iloc->bh);
				/* 修改inode */
				ext3_mark_iloc_dirty(handle, inode, &iloc);
					struct buffer_head *bh = iloc->bh;
					raw_inode->i_atime = cpu_to_le32(inode->i_atime.tv_sec);
					ext3_journal_dirty_metadata(handle, bh);

		ext3_add_nondir(handle, dentry, inode);
			ext3_add_entry(handle, dentry, inode);
				/* 在父目录里找到一个能容纳这个文件的block */
				bh = ext3_bread(handle, dir, block, 0, &retval);
				add_dirent_to_buf(handle, dentry, inode, NULL, bh);
					/* 这个bh当作metadata */
					ext3_journal_get_write_access(handle, bh);
					/* 修改父目录的inode信息 */
					ext3_mark_inode_dirty(handle, dir);
					ext3_journal_dirty_metadata(handle, bh);
			/* 再dirty一次, 似乎没有用? */
			ext3_mark_inode_dirty(handle, inode);

		ext3_journal_stop(handle);

### unmap_underlying_metadata

之前讲过, 同一个磁盘block在内存中可能会存在2个缓存, 但这个函数肯定不是为了处理那些直接对裸设备的操作, 因为它只会对新分配的block才会调用. 试想一下EXT3文件系统的这个过程:

1. 在transaction 1, 修改metadata block n
2. 在transaction 2, 删除block n, 因为该bh还属于transaction 1, 不能执行bforget
3. transaction 1 committed, 设置block n的bh为dirty, 准备回写
4. transaction 2 committed, block n变成free状态
5. get_block重新分配block n为data block, **需要unmap_underlying_metadata**
6. 修改block n, 并写入磁盘
7. 如果没有5的unmap_underlying_metadata, 由于bh为dirty, 写回到磁盘从而覆盖6中正确的数据

如果换成free metadata block n, realloc metadata block n会出现什么状况? 因为第6步写metadata会先进入journal, 所以不会有问题.

# 结语

本文介绍了文件系统的一些基础知识, 希望有助于理解和开发文件系统. 从上层来看, 文件系统并不复杂, 即使是现代的文件系统, 其核心的设计也往往并不太多, 但是想要打造一个稳定可靠的文件系统却不容易, 往往需要几年甚至更久的打磨, 毕竟文件系统是与数据打交道, 不能有丝毫的差错. 细节决定成败.
