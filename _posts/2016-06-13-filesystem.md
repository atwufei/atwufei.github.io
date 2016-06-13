---
layout: post
title: Linux Filesystem Primer
author: JamesW
categories: Kernel
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

虽然能够完成任务, 但却不够好. 表2的条目相对较大, 需要存储数据块的引用, 还包括一些其他的元数据, 这样就会导致一条目录项变得很大, 从而导致一个Block能够只能存储有限数目的文件, 这样对文件的查找是非常不利的, 因为可能需要更多的磁盘访问. 但是如果直接使用文件名去关联2张表的话, 也不是什么好主意, 所以在Unix文件系统里, 每一个文件都会对应到一个数字(index), 通过这个文件系统唯一的编号就可以快速地找出文件所有内容的磁盘位置. 简单地说, inode就是这个index, 再加上其他的一些元数据. inode还有另外的一个小小的好处, 就是可以很容易实现硬链接(hard link).

![]({{ "/css/pics/fs/dir-file-2.png"}})

至于文件内部的组织方式, 不同的文件系统更是千差万别, 可以是树形的, 也可以是简单索引, 甚至还可以让别的文件系统来解析. 一般而言, 文件系统的最小单位是数据块(block), 而一个block往往会包含一个或者多个磁盘的扇区(sector), 扇区是磁盘访问的最小单元.

# EXT2 文件系统

EXT2作为经典的文件系统, 又不失简单高效, 非常适合入门.

## 磁盘分布

想要深入理解EXT2, 了解它的磁盘分布(Disk Layout)是必不可少的. 首先来看看它的整体分布, (本文大量使用 Understanding the Linux Kernel 的图表)

![]({{ "/css/pics/fs/ext2-layout.png"}})

首先整个磁盘或者分区分成一个个的block group, 为了防止文件系统碎片, EXT2会尽量把一个文件分配到同一个Group里面, 保证更高效的访问文件. 从上图可知, 每个block group中只有一个block用作data block bitmap, 每个group的最多能存放 8 * block size * block size字节, 如果block size是4KB的话, 那么一个group可以存放128MB数据.

一个block group包含这些元素:

* super Block: 用于记录整个文件系统的信息. 虽然每个group都包含了一个super block, 但是 并不要求严格统一. 话说即使这些super block都是同步的, 如果group 0 的super block坏了, 是否就可以通过其他的来修复呢? 我们当然是这么期望的, 不过现在好多的disk, 不要说AFA, 即使是普通的SSD盘, 都有可能集成去重功能(Dedup), 也就是说相同的数据在磁盘上可能只有一份拷贝.  也就是说, 上层文件系统想通过冗余来改善系统的稳定性, 下层却把冗余给去重了. 这样看来, 同步倒还不如不同步了, 不同步至少减少了Dedup的可能. 由此可见, 一个好的约定, 理解对方的需求是多么重要, 否则两边都给出了完美的解决方案, 合在一起却不能达到预期的效果.

* group descriptors: 每个group的相应信息, 主要包括data block, inode bitmap和inode table的位置信息, 已经空闲block和inode的个数. 每个group不止包含自己的descriptor, 还有可能包含其他group的descriptor, 也就是说有冗余. 虽然说这个descriptor的数据结构并不大, 但是每一个descriptor都占用了一个block, (如果多个group descriptors共用一个block, 担心修改其中的一个group descriptor会影响到其他的?  不太应该). 对于一个很大的文件系统, 如果每个group都需要保存所有的group descriptors, 这个space overhead会非常大, 所以应该是可配置的, 见descriptor_loc().

* data block bitmap:
* inode bitmap: 使用bitmap来标记对应的block或者inode是否已经分配, bitmap很适合快速地查找和分配空闲空间.

* inode table: inode table顾名思义就是存放inode的地方, 每个inode的大小是一样的, 所以根据inode的号可以很容易就找到对应的inode. inode里面记录了文件相关的元数据, 比如说Owner, Mode, Size, 和各种各样的time, 更重要的是包括了文件block的指针, 通过这些指针就可以找到某个具体offset对应的内容. 中间的某些Block指针为0的情况也是允许的, 甚至变成了一个feature, 也就是hole, 这种情况下文件的size和磁盘的使用是不相等的.

那么inode到底是怎么来描述一个文件的呢? 很多文件系统的教科书上都会有介绍, 而且基本上是跟EXT2一致的. 这种数据结构一定程度上体现了简单有效, 如果是小文件的, 可以很方便地使用12个direct addressing的指针就可以表示出整个文件, 如果更大的文件, 可以通过indirect block来引用. 虽然谈不上完美, 比如每个引用只能索引一个block, 也能较好的解决了问题.

![]({{ "/css/pics/fs/inode.png"}})

EXT2的元数据也就以上几种, 不过还有一种数据比较特殊. 由于文件名的长度各不相同, 为了节省空间, EXT2使用如下变长的结构, 在结构里面通过rec_len来表示该结构的长度. 如果目录是一直增加的, 那实现起来非常简单, 不过怎么支持文件删除, 删除了的空间怎么再次得到利用, 一定程度上增加了目录操作的复杂度, 有兴趣的可以考虑一下下图中inode 67的rec_len. 这个目录结构虽然紧凑, 不过要在其中找一个文件, 却没有什么好办法, 只能从头到尾扫描, 如果目录里面有很多文件或者子目录, 扫描的效率不会太高. 如果目录项根据文件名排好序的话, 检索特定的文件就会非常方便, 很多更现代的文件系统中使用btree来达到这个目的.

![]({{ "/css/pics/fs/dir.png"}})

## 性能优化

了解了以上所说的disk layout, 对EXT2可以说已经有个初步理解. 对于文件系统设计来说, 还有一个很重要的问题就是怎么保证文件访问的性能, 特别是怎么优化磁盘I/O, 虽然上面或多或少已经有所考虑, 但是还有2个方面没有涉及, 特别是data在哪里分配决定了文件系统的性能, 毕竟大量的文件操作是访问data, 而不应该是metadata.

### inode在哪里分配

如果是普通文件, 使用find_group_other()得到新创建文件inode的Block Group.

1. 尽量和父目录放在一起. 如果只考虑文件本身和它的父目录, 这个会有意义吗? 这样的locality会带来读时的优化, 还是写时的优化? 其实并没有. 一般来说, data会和inode同属一个bg (block group), 读文件的时候, 先去读取父目录的data, 这里面会记录文件的inode, 然后从inode里面取得block的信息, 注意inode在bg里是放在开头的, 也就是说读的时候并不能减少seek时间, 甚至相反. 在创建的时候, 因为要同时修改文件的inode和父目录的inode, locality会带来好处? 不过和其他的rule联合起来, 这样做却很容易把有访问locality的文件聚集到一起, 从而提高性能. 如果真有必要弄清楚, 通过trace可以很容易作出判断, 有时候傻傻地想再多还不如动一下手.

2. 尽量把相同目录下的文件放在一起. 假设用户访问文件有locality, 比如多进程编译kernel的时候, 同一个目录的文件会集中访问, 放在一起就可以有效地减少seek的时间 (不只inode, 还有inode对应文件的data block). 注意底下的iosched能对I/O进行排序和合并.

3. 如果父目录所在的bg已经满了, 尽量岔开不同的目录. 如果还把这些目录的文件放在一起, 根据之前的经验, 新的bg也会有更大的概率便满. 这其实和2是一致的. 对应的代码如下:
	group = (group + parent->i_ino) % ngroups;
	for (i = 1; i < ngroups; i <<= 1) {}

对于新创建的目录, 老的方法find_group_dir()并没有讲究太多, 只是选出一个free inodes和free blocks相对较多的block group. 新方法find_group_orlov()考虑的更多一些, 不再详述. 如果有兴趣, 可以阅读一下这篇文章 [Locality and The Fast File System](http://pages.cs.wisc.edu/~remzi/OSTEP/file-ffs.pdf)

### data在哪里分配

data block的分配可能显得更加重要, 访问data block一般来说比metadata多得多, 访问时locality更强, 碎片导致的性能问题也会更大. data block, 包括indirect block, 都在ext2_get_block()里分配, 更具体点就是ext2_find_near():

1. 尽量靠近前面的block
2. 尽量靠近上一层indirect block
3. 尽量放在inode的bg

除了这些努力之外, 另一个有效的办法就是预先reserve一片连续的磁盘空间, 虽然get_block()会尽量就近分配, 但有时却无奈其他文件分配的干扰. 比如文件A已经分配了block n, 本来正打算分配下一个block, 最好的选择就是Block n+1, 可是文件B恰好就已经分配了Block n+1. 为了避免这种情况, 预先reserve [n, n+window]是个比较好的选择. 更妙的是, 这个reserve并不需要浪费物理磁盘的空间, reserve整个是在内存中完成的, 当没有人再打开该文件的时候, 就可以释放reserve的区间. 一般来说, 我们会一次增加大量的数据, 而不是每打开一次增加一两个数据块, 所以这种内存reserve方式可以很好的工作.

# 文件的访问

