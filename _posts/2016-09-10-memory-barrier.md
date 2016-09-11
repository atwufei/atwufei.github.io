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

一直简单地认为x86是strong memory model, 软件需要介入的地方不多, 但是更具体点的就不得而知了. 直到最近同事在debug的时候发现了一个memory barrier的问题, 所以才重新的重视起来. 虽然这个bug并不是我们自己解的, 只不过刚好存在于我们的Kernel版本里, 但是还是很有助于理解这个问题.

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

结果就是CPU1 observe的顺序与CPU0上的程序顺序是不一致的, 即使是X86这种strong ordered memory model, 上面这种情况#StoreLoad也是会发生的. 这种observed & program不一致性只有在多CPU上才会出问题, 单CPU上如果前后两个操作有依赖关系, 自然不会乱序；如果它们并不相关, 则不会引起错误.

**只有observe的顺序是跟最后结果相关的.**

### Fix

首先需要说明的是, Memory Barrier只能管理本CPU的状态, 并不能强制其它CPU的行为. 对于smp_mb, 在 Paul E. McKenney [Is Parallel Programming Hard, And, If So, What Can You Do About It?](http://kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook-1c.2016.07.31a.pdf) 的描述是:

“memory barrier” that orders both loads and stores. This means that loads and stores preceding the memory barrier will be committed to memory before any loads and stores following the memory barrier.

这里需要注意的是:

* "committed to memory"表示observable to other CPU, 但是并不表示other CPU 已经observed这些操作, 也不表示other CPU会按这个顺序observe这些操作, 除非other CPU也使用了相应的barrier, 否则也就不用双方都使用相应的barrier了.

* "memory barrier"也不保证前面的操作已经"committed to memory", 它只是要求保证它之前和之后内存操作的**顺序**, 也就是说这个操作理论上可以完全不通知其它CPU, 在本CPU就可以完成. 甚至在memory barrier完成时前面的memory operations并不要求完成.

可以简单地认为memory就是一个global shared memory/cache, committed to memory的意思:

* 对于store来说, 就是把store提交到这个memory上了, 而不是只存在于CPU的local cache比如store buffer上. 但是这个还是不能保证其它的CPU observe到这个store, 其它CPU需要主动去pull这个结果.
* 对于load来说, 就是去pull global memory里面的最新内容, 更新自己可能存在的过时的内容.

具体到这个问题上面, 因为CPU0和CPU1上面都使用了smp_mb: 

* 如果CPU0上的smp_mb后执行, 则CPU1上prepare_to_wait; smp_mb已经执行, prepare_to_wait的结果已经commited to memory, CPU0上的smp_mb会pull这个结果, 所以会调用wake_up, 因为已经observe CPU1上的prepare_to_wait, wake_up将会确保CPU1上的schedule能够退出, 从而重新执行这个循环, 由于它prepare_to_wait调用了smp_mb, 所以CPU0上@cond = true必然被observe, 从而break.

* 如果CPU1上的smp_mb后执行, 则CPU0上的@cond = true必然在CPU1被observe, 从而CPU1可以break.

## Memory Barrier原理

Memory Barrier难以理解的一个原因是大部分文档都比较晦涩. 当接口不是很清晰的时候, 如果能够理解里面的实现就很有必要.

