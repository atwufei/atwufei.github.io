---
layout: post
title: How to write file in Linux
author: JamesW
categories: Ext4 filesystem
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content
{:toc}

Linux系统访问文件有多种方式, 对于写来说, 最终会修改这两个地方:

* data: 文件本身的内容, 也是写操作的根本所在
* metadata: 文件系统的管理数据, 以及文件的管理数据(inode及相关)

## Linux文件系统

对于Ext系列文件系统来说, 文件系统的管理数据参考这幅图, 主要包括几个bitmap:

![fs meta]({{ "/css/pics/file-write/ext.png"}})

inode的信息可以分成2个部分:

* 文件附带的属性, 读取文件内容的时候并不依赖它们 (nonvital-attr)
  * file mode
  * uid, gid
  * atime, ctime, mtime
  * ...

* 读取文件内容时依赖的信息 (vital-attr)
  * size
  * i\_block[], block是文件系统的最小管理单元. 对于Ext2/3, 文件的每一个block都需要单独的索引,
对于Ext4, 可以用extent来描述多个连续的block. 这个也用来描述extent相关的信息.
  * ...

![ext4 extent]({{ "/css/pics/file-write/ext4-extent.png"}})

## 写操作分类

根据上面的分析, 我们可以把write分成2类:

* 覆盖写: 这种写操作不改变inode的vital-attr, 不分配空间, 也不会改变文件系统级的metadata.
	比如已经有一个2MB的文件, 现在只是去修改其中的某一部分.
	如果应用不关心nonvital-attr, 比如mtime, 则只要保证data的落盘.

* 非覆盖写: 这种写操作会触发metadata的改变, 特别是vital-attr. 如果这些metadata的修改没有写盘,
	则data即使落盘也不能够读取. 为了保证data能够访问, 需要在write操作后调用fsync类函数.

## 常用写文件的方法

要理解怎么写文件, 先看看系统里面都有哪些东西是跟文件相关.

![file i/o]({{ "/css/pics/file-write/fileio.png"}})

从上图中可以看出, Linux系统中文件内容主要存在以下地方:

* 磁盘文件, 文件只有写入到磁盘才是非易失的, 当然磁盘本身又可能使用disk write cache, 掉电同样可能丢失数据.
* page cache 用来缓存文件的内容, write操作只需要把data写入page cache即可返回, read操作也可以通过cache来加快访问.
	如果需要persistent data, 最终cache还是需要写入磁盘的, 或者是系统后台自动flush, 或是用户程序主动sync.
* buffer cache 用来缓存磁盘的内容, 它的主要来源是文件系统的metadata, 以及直接访问裸设备.
	如果把裸设备当成文件, 则可以简单认为buffer cache是这个裸设备的page cache. 因为page cache和buffer cache的存在,
	同一份磁盘数据在内存中可能存在2份copy, 一份是普通文件访问产生的page cache, 一份是访问裸设备产生的buffer cache,
	绝大部分情况, kernel不会去同步这2份copy, 用户最好避免这样使用.
* libc buffer, 如果不直接通过syscall访问文件, 而是通过libc的接口, 则libc本身会有一个buffer, 这样调用fwrite()并不会
	每次都触发一个syscall, 而且libc的可移植性更好. 如果程序只跑在Linux上, 并且对文件I/O比较熟悉, 完全可以自己来决定
	是否使用buffer, 怎么使用这个buffer, 多大的buffer, 然后直接通过syscall进行I/O. 下面不再讨论这个.

### page cache flush

数据放在page cache有丢失的风险, 所以需要适时的把page cache flush到磁盘上.
page cache同时又能起到合并写, 集合写的功能, 如果flush得太频繁可能会导致更多更离散的磁盘I/O.
在Linux系统中一般有2个地方控制flush的频率:

* 文件系统本身提供的机制, 比如Ext3/4, mount option *commit*控制了journal写回的间隔.
  注意, 虽然手册里面这个参数同时控制data和metadata, 但其实它只是控制了journal的写回.
  在data=ordered的情况, 因为metadata会做journal, 同时data需要在metadata之前写入磁盘,
  这个参数可以控制data和metadata的写回. 在data=writeback的情况下, 没有对data和metadata
  写盘的顺序有要求, 所有commit参数只能控制metadata的刷盘时间, 默认情况下commit的间隔设置为5s.
  可以通过iostat观测这个命令的效果:

		# dd if=/dev/zero of=data bs=100M count=1

* 虚拟内存子系统在后台也会定时的把dirty page刷回磁盘, 文件系统本身并不需要额外的操作,
  比如Ext2就没有自己的后台线程, 而Ext3/4如果设置为data=writeback, flush dirty *data* page同样通过这个控制.
  这个时间通过/proc/sys/vm/下的2个文件控制:

		- dirty_expire_centisecs, dirty page expire的时间, ubuntu上现在默认设置为15s
		- dirty_writeback_centisecs, 后台线程wakeup的interval, ubuntu上默认15s

当然内核有权利提前写回, 比如系统中有太多的dirty page的时候, 及时写回可以避免大量数据的丢失.

### regular i/o

最普通的方法, 文件读写经过page cache, 绝大部分应用程序都会使用这种方式.

	fd = open(file_name, O_RDWR);
	write(fd, buf, buf_size);

#### concurrency i/o

并发写不同文件不是问题, 那么写同一个文件是否能够并发呢? 我们先来看看ext3_file_operations->aio_write,

	ssize_t generic_file_aio_write(struct kiocb *iocb, const struct iovec *iov,
			unsigned long nr_segs, loff_t pos)
	{
		struct file *file = iocb->ki_filp;
		struct inode *inode = file->f_mapping->host;
		ssize_t ret;

		BUG_ON(iocb->ki_pos != pos);

		mutex_lock(&inode->i_mutex);
		ret = __generic_file_aio_write(iocb, iov, nr_segs, &iocb->ki_pos);
		mutex_unlock(&inode->i_mutex);

		if (ret > 0 || ret == -EIOCBQUEUED) {
			ssize_t err;

			err = generic_write_sync(file, pos, ret);
			if (err < 0 && ret > 0)
				ret = err;
		}
		return ret;
	}

buffer i/o和direct i/o的逻辑都在\_\_generic\_file\_aio\_write中, 
通过i\_mutex完全串行起来, EXT4也是类似逻辑.

所以对于ext3/ext4来说, 同一文件的buffer i/o和direct i/o是完全串行化的.
对于buffer i/o这并不是什么大问题, 因为write不过是内存拷贝, 但是对于direct i/o来说,
是需要写盘的, 串行对性能可能会有很大影响. 运行这个简单的fio测试验证, 通过iostat观测
avgqu-sz根本没有并行:

	[test]
	filename=datafile
	filesize=2g
	ioengine=sync
	bs=4k
	rw=randwrite
	direct=1
	numjobs=16



### O\_DIRECT

数据库这类的应用, 更倾向于自己来管理cache, page cache不只没有帮助, 反而浪费了额外的cpu和memory资源.
Linux像其他的OS一样, 提供了direct i/o机制. direct i/o可以把应用程序的buffer直接dma到存储设备,
从而直接bypass page cache.

不过direct i/o并不像看起来这么简单, 首先file offset, buf, buf\_size均有对齐要求.

	fd = open(file_name, O_RDWR | O_DIRECT);
	write(fd, aligned_buf, aligned_buf_size);

更麻烦的是, direct i/o只写data, 但是文件还有metadata, 特别是非覆盖写的情况, 文件系统需要分配磁盘空间.
但是direct i/o在返回的时候并不保证metadata的落盘. 这样即使direct i/o成功把data写到磁盘了,
系统如果这个时候crash, 依然不能读取data.

direct i/o跟buffer i/o同时使用时

* direct i/o是同步的, direct i/o返回时, 应用程序的buffer就可以reuse了
* direct i/o并不保证刷disk write cache
* **_direct i/o并不保证bypass page cache_**, 也就是说它返回的时候数据可能还在内存中.

	https://ext4.wiki.kernel.org/index.php/Clarifying_Direct_IO%27s_Semantics

	Given that with an allocating write, an explicit fsync(2) (or write with
	O_SYNC/O_DSYNC) is required, there doesn't seem to be much point in waiting
	until the data I/O is complete if the O_DIRECT write has fallen back to using
	buffered I/O --- after all, if the data has been copied into the page cache,
	the data buffered passed into the write(2) system call can be safely reused for
	other purposes, so it may be that the kernel should be allowed to return as
	soon as the data has been copied into the page cache.

	From a specification point of view, the fact that allocating writes can fall
	back to buffered I/O should be documented, and that any file system control
	data associated with the block I/O will not be synchronously committed unless
	the application explicitly requests this via fsync(2) or O_SYNC.
readahead


### O\_DIRECT + O\_SYNC

### direct i/o + aio + o\_sync
commit 9c225f2655e36a470c4f58dbbc99244c5fc7f2d4

### io concunrrency (i_mutex)
mmap:
* 之后的读写不需要syscall
* 写一个byte就需要读入整个block, write()则可以直接覆盖
* 大块读, 怎么预读, 性能问题, 需要靠读触发
* sigbus, mmap成功, 但是读写时失败, 比如mmap(len), 但是文件却没有那么大
* transaction, 每个mmap的写是独立的, journal用不上. mmap的读写本身不能改变大小, 也就无所谓transaction?
* mmap脏页写回, msync
* mmap with holes, 没什么差别

每次对mmap的dirty page写回之后, 都会重新wrprotect这个page, 之后的访问重新触发page fault

	write_cache_pages
	ext4_page_mkwrite // 并没有journal data, 所以写回并不受commit=interval控制, 即使data=ordered
	  handle = ext4_journal_start(inode, ext4_writepage_trans_blocks(inode));
	  ret = __block_page_mkwrite(vma, vmf, get_block);

	dirty_expire_centisecs/dirty_writeback_centisecs
commit=interval控制的是journal的写回, 在data=ordered的情况下, 间接控制了data的写回,
但是对于data=writeback, 就不能控制data的写回, 这些dirty data(page)的写回则靠dirty_expire_centisecs它们

http://www.datarecoverytools.co.uk/2009/11/16/learn-more-about-ext4/


## O_DIRECT | O_ATOMIC

directio, metadata
