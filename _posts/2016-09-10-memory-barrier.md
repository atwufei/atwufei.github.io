---
layout: post
title: Memory Barrier
author: JamesW
categories: Kernel
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

## Memory Barrier的一个例子

### Bug

一直简单地认为x86是strong memory model, 软件需要介入的地方不多, 但是更具体点的就不得而知了. 直到最近同事在debug的时候发现了一个memory barrier的问题, 所以才重新的重视起来. 虽然这个bug并不是我们自己解的, 只不过刚好存在于我们的kernel版本里, 但是还是很有助于理解这个问题.

	/* Use either while holding wait_queue_head_t::lock or when used for wakeups
	 * with an extra smp_mb() like:
	 *
	 *      CPU0 - waker                    CPU1 - waiter
	 *
	 *                                      for (;;) {
	 *      @cond = true;                     prepare_to_wait(&wq, &wait, state);
	 *      smp_mb();                         // smp_mb() from set_current_state()
	 *      if (waitqueue_active(wq))         if (@cond)
	 *        wake_up(wq);                      break;
	 *                                        schedule();
	 *                                      }
	 *                                      finish_wait(&wq, &wait);
	 *
	 * Because without the explicit smp_mb() it's possible for the
	 * waitqueue_active() load to get hoisted over the @cond store such that we'll
	 * observe an empty wait list while the waiter might not observe @cond.
	 *
	 * Also note that this 'optimization' trades a spin_lock() for an smp_mb(),
	 * which (when the lock is uncontended) are of roughly equal cost.
	 */
	static inline int waitqueue_active(wait_queue_head_t *q)
	{
		return !list_empty(&q->task_list);
	}

问题出现在调用这个函数的地方:

	unlock_new_inode(struct inode *inode)
	{
		lockdep_annotate_inode_mutex_key(inode);
		spin_lock(&inode->i_lock);
		WARN_ON(!(inode->i_state & I_NEW));
		inode->i_state &= ~I_NEW;               // <-- @cond
		smp_mb();                               // <-- missing
		wake_up_bit(&inode->i_state, __I_NEW);  // <-- waitqueue_active()
		spin_unlock(&inode->i_lock);
	}

如果CPU0上没有加上smb_mb的话, waitqueue_active的检查可能在@cond赋值之前完成:

* 如果CPU0 waitqueue_active返回0 - 或者是prepare_to_wait根本没执行, 或者CPU0并没有observe看到prepare_to_wait往waitqueue的写操作, 总之wake_up没有被调用
* 如果CPU1 if(@cond) 返回0进入schedule - 或者是CPU0 @cond = true还没执行, 或者是CPU1还没有observe到该事件, 这样CPU1就会进入到schedule而没人唤醒的地步

结果就是CPU1 observe的顺序与CPU0上的程序顺序是不一致的, 即使是X86这种strong ordered memory model, 上面这种情况#StoreLoad也是会发生的, 这应该也是WB内存上唯一的一种情况(除了SSE等特殊操作). 这种observed & program不一致性只有在多CPU上才会出问题, 单CPU上如果前后两个操作有依赖关系, 自然不会乱序；如果它们并不相关, 则不会引起错误.

**只有observe的顺序是跟最后结果相关的.**

### Fix

首先需要说明的是, memory barrier只能管理本CPU的状态, 并不能强制其它CPU的行为. 对于smp_mb, 在 Paul E. McKenney [Is Parallel Programming Hard, And, If So, What Can You Do About It?](http://kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook-1c.2016.07.31a.pdf) 的描述是:

“memory barrier” that orders both loads and stores. This means that loads and stores preceding the memory barrier will be committed to memory before any loads and stores following the memory barrier.

这里需要注意的是:

* "committed to memory"对于store来说表示observable to other CPU, 但是并不表示other CPU 已经observed这些操作, 也不表示other CPU会按这个顺序observe这些操作, 除非other CPU也使用了相应的barrier, 否则也就不用双方都使用相应的barrier了. 对于load来说, 比如load speculate, 如果有改变, 那么barrier之后就需要重新读入该值. (虽然说指令issue/complete都是按程序顺序完成, 但如果使用的是speculate的old data, 那么看起来就是乱序.)

* "memory barrier"也不保证前面的操作已经"committed to memory", 它只是要求保证它之前和之后内存操作的**顺序**, 也就是说这个操作理论上可以完全不通知其它CPU, 在本CPU就可以完成. 甚至在memory barrier完成时前面的memory operations并不要求完成.

可以简单地认为memory就是一个global shared memory/cache, committed to memory的意思:

* 对于store来说, 就是把store提交到这个memory上了, 而不是只存在于CPU的local cache比如store buffer上. 但是这个还是不能保证其它的CPU observe到这个store, 其它CPU需要主动去pull这个结果.
* 对于load来说, 就是去pull global memory里面的最新内容, 更新自己可能存在的过时的内容.

具体到这个问题上面, 因为CPU0和CPU1上面都使用了smp_mb: 

* 如果CPU0上的smp_mb后执行, 则CPU1上prepare_to_wait; smp_mb已经执行, prepare_to_wait的结果已经commited to memory, CPU0上的smp_mb会pull这个结果, 所以会调用wake_up, 因为已经observe CPU1上的prepare_to_wait, wake_up将会确保CPU1上的schedule能够退出, 从而重新执行这个循环, 由于它prepare_to_wait调用了smp_mb, 所以CPU0上@cond = true必然被observe, 从而break.

* 如果CPU1上的smp_mb后执行, 则CPU0上的@cond = true必然在CPU1被observe, 从而CPU1可以break.

## Memory Barrier原理

Memory barrier难以理解的一个原因是大部分文档都比较晦涩. 当接口不是很清晰的时候, 如果能够理解里面的实现就很有必要, 上面提到的Paul McKenney的书很有帮助.

### 为什么有Memory Barrier

试想一下, 如果每个CPU都直接访问一片shared memory/cache, 所有的内存访问都是全局可见的, 也就用不着memory barrier. 之所以不这么设计, 只有一个原因, 慢.

#### Cache Coherence

真实的CPU设计, 因为每个CPU都有单独的cache, 需要使用某种协议来维护这些cache的coherence, 比较典型的就是MESI. MESI的核心思想就是对于一个特定的cache line, 只能有一个CPU有写权限, 从而达到coherence的目的. 也就时说, 整个cache系统看到的Store是有一个顺序的, 这个顺序也决定了所有CPU看到的最终状态, 但是每个CPU看到的临时顺序依然没有保证. MESI的状态图并不是太复杂, 详情可以参考Paul的那本书.

#### 优化Cache访问

Cache的优化主要从store的角度, 对于Load来说总得等到数据ready才能进行计算, 也就是说load一般是个sync操作, 但是对于store并没有这个要求, 为了减少store的等待时间, 可以尽可能地把store操作做成async. 这主要有2个方法:

* Store Buffer - 数据写入store buffer即可返回, 并不需要等待invalidate ack, 甚至不发送invalidate msg (比如前面有store;mb, 并且store并没有收到invalidate ack)
* Invalidate Queue - 但是数据总要从store buffer写入cache, 从而产生invalidate message, 而目标CPU可以把这个message放入它的invalidate queue里面, 并直接ack.

所以一个可能的CPU架构可能如下:

![cpu arch]({{ "/css/pics/barrier/cpu-arch.png"}})

可以简单地认为smp_wmb操作的是store buffer, 而smp_rmb操作的是invalidate queue. 不过这并不表示

	smp_wmb + smp_rmb == smp_mb (X)

比如x86上smp_wmb和smp_rmb都是nop, 不过smp_mb却是mfence.

按照之前"commited to memory"的说法, smp_mb需要drain store buffer 和 invalidate queue, 而smp_wmb并不需要drain store buffer, 只是需要保证barrier前后store之间observable to others的顺序, 比如可以mark之前store buffer的entry, 而block住unmarked store buffer entry去写cache(observable to others).

理解memory barrier的关键就是理解这两个顺序:

* observable order to others - push
* observed order of others - pull

**只有两边push和pull合作, 才能保证逻辑的正确.**

### 怎么使用Memory Barrier

举个Documentation/memory-barrier.txt里的简单例子

	CPU 1			CPU 2
	=======================	=======================
		{ A = 0, B = 9 }
	STORE A=1
	<write barrier>
	STORE B=2
				LOAD B
				// read barrier required
				LOAD A

如果CPU2上LOAD B=2, A的值并不确定, A可能存在于CPU2的invalidate queue上还未处理.

	CPU 1		      CPU 2
	===============	      ===============================
		{ x = 0, y = 0 }
	r1 = READ_ONCE(y);
	<general barrier> // <--
	WRITE_ONCE(x, 1);     if (r2 = READ_ONCE(x)) {
			         <implicit control dependency>
			         WRITE_ONCE(y, 1);
			      }

	assert(r1 == 0 || r2 == 0);

这里倒跟store buffer和invalidate queue没关, 但是必须保证WRITE_ONCE在READ_ONCE之后执行.

## 总结

这里还有很多东西没有涉及, 比如compiler barrier, acquire/release的语义, smp_read_barrier_depends, mandatory mb(without smp\_ prefix)等等, 不过了解了CPU的架构之后, 理解起来会相对容易很多.
