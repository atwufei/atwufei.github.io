---
layout: post
title: Slow fsync
author: JamesW
categories: Kernel Storage
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

## 问题: fsync有时超过30s

fsync对于Data Integrity的重要性不言而喻, 在我们的系统中fsync主要来源于两个地方:

* syslogd, 每条log之后都会调用fsync, 如果记录的log过多的话, 对fsync的性能有很高要求. 当然每条log之后都fsync是可以讨论的, 一个更直接的方法就是减少syslog的数量, 这个基本上是可以做到的.
* SQLite, 我们使用SQLite保存registry, registry里面不只是一些静态的configs, 也包括一些runtime的状态, 有些SQLite的访问甚至出现在data path上. 如果这些SQLite操作不能在规定时间完成, 我们会主动关闭业务逻辑, 所以SQLite的访问, 特别是写访问的速度对于我们至关重要. SQLite的写操作可想而知需要调用fsync, 当大量的fsync出现在系统中时, 对I/O的性能就是很大的挑战.

在继续讨论之前, 先来看看简化过的I/O stack:

![I/O stack]({{ "/css/pics/slow-fsync/io-stack.png"}})

* 最底层是HDD组成的shelf
* 中间层通过RAID 6把底层的HDD组合起来
* 最上层的文件系统创建在RAID的分区上面, 都使用EXT3

在系统的正常运行过程中, 我们经常可以观测到fsync系统调用需要30s甚至更久时间才能完成, 这显然是不能接受的.

## 重现问题

通过系统测试去重现问题只会事倍功半, 即使重现了信息也不一定能够收集全. 因为已经确定在fsync的时候会出现问题, 那么我们可以简单地写一个fsync的测试用例, 之所以没使用fio等现成的工具, 是因为想要收集一些更细粒度的信息, 比如每个fsync花费的时间等等.

简单的测试发现, 正常的fsync并不需要多久, 更不用说30s, 这也是符合预期的. HDD再慢, 随机读写应该也有100 iops左右, 如果fsync的数据不多的话应该没有问题. 那么问题出在哪里呢? 在调查的过程中, 发现当出现slow fsync现象之前, 经常可以看到大文件的产生, 比如某个log文件被压缩了, 这个通过文件的时间戳就很容易发现, 只要有针对性地去找.

那么在重现的时候后台增加一个dd的进程, 果然一定程度重现了该问题.

## 解决方法part1: 减少fsync的依赖

在我的博客 [Linux Filesystem Primer](http://wufei.org/2016/06/13/filesystem/) 里讲到EXT3 fsync时, 由于FIFO journal的原因, 对于data=ordered模式下的fsync, 它不只是flush自己的data, 还需要flush之前和同一个journal里面其他文件的data page, 这就引入了[无谓的]dependency. 就像我们的系统里, 当同时有一个大文件写的时候, 由于这种依赖关系极大地延长了fsync的时间. 这个问题可以通过data=writeback来解决, fsync的performance提升了, 不过Integrity呢? 按理说是没有什么问题的, 不过企业级产品一般都比较保守.

EXT4通过delayed allocation解决了这个问题, 既然磁盘都还没有分配, 当然也就谈不上flush的问题了. 经过测试, 如果disable delayed allocation, 这个测试结果跟EXT3就基本一致了. 当然delayed allocation因为长时间没有flush, 导致数据丢失的可能性增大.

当然我们可以使用一个文件系统来单独存放registry, 这其实是个可行的方案, 从根本上解决了fsync在文件之间的依赖关系, 而不用去管具体使用哪个文件系统.

## 解决办法part2: 减少disk I/O竞争

除了上面提到的硬性依赖条件, 在做I/O时还需要考虑不同I/O之间的竞争, 也就是多个I/O的优先级. 即使2个文件系统完全独立, 但也要竞争相同的I/O资源.

* 一种方法就是直接提高某个具体文件系统的I/O优先级, 当然这需要更改iosched的代码.
* 另一种方法就是提高某种具体操作的I/O优先级, Linux kernel已经提供这种操作, 只要是REQ_SYNC的操作, 在cfq iosched里会得到更高的优先级. 这里面包含2个方面, 一是在最下层需要知道这个flag, 也就是说我们自己的RAID需要把这个flag传下去, 另一个就是我们自己的iosched需要处理这种情况.

## 解决办法part3: 增加底层I/O带宽

另一个可以改进的地方是, 如果底层的I/O带宽大幅增加了以后, 这个问题是不是能够缓解, 答案是肯定的. 比如说:

* 把registry所在文件系统放到快速的磁道上, 即使同一磁盘, 不同的地方throughput也能相差一倍左右
* 优化I/O stack的性能, 比如RAID是不是有改进空间
* 使用不同的介质, 把HDD换成SSD

不过一两倍其实并不一定能够起太大的作用. 这个角度并没有上面2个作用大.

当然还有一种情况就是, 底层I/O带宽大幅降低, 比如RAID需要rebuild, 怎样分配更多的资源给fsync?哪个重要?

## 回顾问题

这个问题存在于我们的系统多年, 虽然说中间得到了一定程度的改善, 却并没有从根本上解决. 其实通过对整个I/O stack分析, 给出一个相对完整的解决方案也并不复杂. 不过slow fsync并不会因此就消失了, 毕竟它还是会依赖于底层的infrastructure, 但是我们还能做的更好吗?

