---
layout: post
title: Linux Scheduler
author: JamesW
categories: Kernel
---

转载须注明出处：[www.wufei.org](http://www.wufei.org)

* content 
{:toc}

## Framework

进程调度简单地说无非是换入换出, 但是如何选择换入换出却是一门学问.
除了需要考虑用户主动设置的优先级之外, 另一个不能忽视的问题就是交互性.
交互性高的程序比如文本编辑器, 需要更快的响应, 交互性低的程序比如程序编译,
则没有这方面的需求. 从最初的简单分时系统, 到O(1)调度器中引入大量复杂(易错的)
的启发式算法, 在进化到如今的CFS算法, 交互性终于得到一个相对满意的解决.
当然, 调度器需要考虑的问题远远不止这些, 比如CPU之间的Load Balance,
NUMA的考量, 更加精细的控制比如Task Cgroup, cpuset等等, 都需要调度器去解决.
所有这些东西综合起来让Linux的Scheduler代码变得非常复杂, 再加上Performance
Tuning, 大大降低了代码的可读性. 为了方便理解, 我们可以一个一个Feature去理解,
尽量避免Context Switch.

不过万变不离其宗, 调度器需要解决的还就是换入和换出两件事, 只是有的解决的好,
有的比较原始一点.

*以下对进程(Process)和任务(Task)不做区别, 都认为是Scheduler调度的最小实体.*

### State

根据状态, 任务一般可以简单地分为两种

* 可以运行的. 在Linux上的状态是TASK_RUNNING. 这种状态的任务是可供调度器
	调入(Sched In)的, 当然还有一个特殊的TASK_RUNNING任务, 那就是正在CPU
	上运行的任务.

* 不能运行的. 这类进程一般都是在等待某种资源或者条件, 在这些条件不满足的情况下
	是不能继续进行的. 这种进程的状态一般是TASK_INTERRUPTIBLE或者TASK_UNINTERRUPTIBLE,
	唯一区别就是TASK_UNINTERRUPTIBLE不能接收信号(Signal), 最直接的效果就是Kill不掉.
	当然还有其他的状态, 比如TASK_KILLABLE. 这类进程是不会挂在Runqueue上的.

### Runqueue

作为一个多任务系统, 系统中肯定会出现多个TASK_RUNNING状态的进程. 要把这些任务组织起来,
肯定需要某种数据结构. 不管是List也好, Tree也好, 还是Queue或者其他的, 在这里统一称之为
Runqueue.

在SMP或者NUMA架构下, 因为系统中有多个CPU, 每个CPU独立执行, 所以在每个CPU上都有自己的
Runqueue. 比如在2个CPU的情况下, 运行Crash的runq命令会输出这些:

    crash> runq
    CPU 0 RUNQUEUE: ffff8800090436c0
      CURRENT: PID: 588    TASK: ffff88007e4877a0  COMMAND: "udevd"
      RT PRIO_ARRAY: ffff8800090437c8
         [no tasks queued]
      CFS RB_ROOT: ffff880009043740
         [118] PID: 2110   TASK: ffff88007d470860  COMMAND: "check-cdrom.sh"
         [118] PID: 2109   TASK: ffff88007f1247a0  COMMAND: "check-cdrom.sh"
         [118] PID: 2114   TASK: ffff88007f20e080  COMMAND: "udevd"

    CPU 1 RUNQUEUE: ffff88000905b6c0
      CURRENT: PID: 2113   TASK: ffff88007e8ac140  COMMAND: "udevd"
      RT PRIO_ARRAY: ffff88000905b7c8
         [no tasks queued]
      CFS RB_ROOT: ffff88000905b740
         [118] PID: 2092   TASK: ffff88007d7a4760  COMMAND: "MAKEDEV"
         [118] PID: 1983   TASK: ffff88007e59f140  COMMAND: "udevd"
         [118] PID: 2064   TASK: ffff88007e40f7a0  COMMAND: "udevd"
         [115] PID: 2111   TASK: ffff88007e4278a0  COMMAND: "kthreadd"

因为只有在Runqueue上的任务才能被调用, 所以任务的调入调出其实就是把任务放入和移出Runqueue
的过程.

### schedule()

作为调度器的同名函数, schedule()显然是首先需要关注的. 这个函数主要解决了进程
调出CPU的过程, 当然它也解决了进程调入CPU的过程, 不过调入CPU并不是Scheduler的
重点, Scheduler关于调入的主要问题在挂入Runqueue和从Runqueue选择Next进程, 至于
选出Next怎么把它调入CPU是相对简单的问题.

*为了减少篇幅, 尽量只提取相应的代码, 而不是完整代码.*

	static void __sched __schedule(void)
	{
		preempt_disable();
		cpu = smp_processor_id();
		rq = cpu_rq(cpu);
		prev = rq->curr;
		raw_spin_lock_irq(&rq->lock);

		if (prev->state && !(preempt_count() & PREEMPT_ACTIVE)) {
			deactivate_task(rq, prev, DEQUEUE_SLEEP);
			prev->on_rq = 0;
		}

		next = pick_next_task(rq);
		if (likely(prev != next)) {
			context_switch(rq, prev, next); /* unlocks the rq */
		}
	}

简单地说, \_\_schedule()主要有三步:

1. deactivate_task
2. pick_next_task
3. context_switch

这里需要特别注意的是deactivate_task()前面的条件:

* prev->state != TASK_RUNNING, 也就是说RUNNING的进程进入schedule()是不会从Runqueue中移除的,
	那么调度器还是可以继续调入这个进程, 并不需要Wakeup的过程.
* !(preempt_count() & PREEMPT_ACTIVE), 进程如果是被Kernel Preempted, 同上. 这个没什么可说的,
	进程被抢占当然不能从Runqueue移除, 否则不能重新被调入.

### ttwu()

任务唤醒, 也就是把任务重新加入到Runqueue里面去. 基本上所有的任务唤醒都是通过try_to_wake_up()
来实现.

	try_to_wake_up(struct task_struct *p, unsigned int state, int wake_flags)
	{
		raw_spin_lock_irqsave(&p->pi_lock, flags);
		if (!(p->state & state))
			goto out;

		cpu = task_cpu(p);
		if (p->on_rq && ttwu_remote(p, wake_flags))
			goto stat;

		cpu = select_task_rq(p, p->wake_cpu, SD_BALANCE_WAKE, wake_flags);
		if (task_cpu(p) != cpu) {
			wake_flags |= WF_MIGRATED;
			set_task_cpu(p, cpu);
		}

		ttwu_queue(p, cpu);
	stat:
		ttwu_stat(p, cpu, wake_flags);
	out:
		raw_spin_unlock_irqrestore(&p->pi_lock, flags);
	}

* 首先, 如果进程是TASK_RUNNING的状态, 函数就直接返回了, 也就是说根本没有或者说根本不需要唤醒.
	一个常见错误就是sleep_on()因为这个丢失wakeup event, 后面会有更详细的分析.
* p->on_rq成立的话, p必定不在当前Wakeup这个CPU上, 因为!RUNNING状态的任务如果在Wakeup CPU上的话,
	调用schedule()的时候就on_rq=0. ttwu_remote()工作相对简单, 就是把p的状态设置成RUNNING就好了,
	这样schedule()就不会deactivate_task(), 也就不会发生Wakeup Event丢失的情况了.
	需要注意的是, schedule()中on_rq=0在rq->lock加锁的状态完成的, 而__task_rq_lock()也对其加锁.

	* 如果ttwu_remote()能获得该锁幷且偏p->on_rq=1, 则p还没有调用到schedule()的主要逻辑,
		这个时候设置RUNNING状态就能保证p不会从Runqueue删除.
	* 否则p已经被删除, 需要执行正常的Wakeup操作.

			static int ttwu_remote(struct task_struct *p, int wake_flags)
			{
				struct rq *rq;
				int ret = 0;

				rq = __task_rq_lock(p);
				if (p->on_rq) {
					ttwu_do_wakeup(rq, p, wake_flags);
					ret = 1;
				}
				__task_rq_unlock(rq);

				return ret;
			}

* 否则, 通过select_task_rq()给新唤醒的进程p设置CPU, 这个时候是可能切换到不同CPU上去的.
	最后通过ttwu_queue()最终把p加入到对应CPU的Runqueue上. 值得一提的是, 本来ttwu_queue()
	代码逻辑上是很简单的, 但是如果是Remote CPU的话, Performance却不行, 因为这涉及到Remote
	Access, 在Commit 317f3941中进行了优化.

		static void ttwu_queue_remote(struct task_struct *p, int cpu)
		{
			if (llist_add(&p->wake_entry, &cpu_rq(cpu)->wake_list))
				smp_send_reschedule(cpu);
		}

		static void ttwu_queue(struct task_struct *p, int cpu)
		{
			struct rq *rq = cpu_rq(cpu);

		#if defined(CONFIG_SMP)
			if (sched_feat(TTWU_QUEUE) && !cpus_share_cache(smp_processor_id(), cpu)) {
				sched_clock_cpu(cpu); /* sync clocks x-cpu */
				ttwu_queue_remote(p, cpu);
				return;
			}
		#endif

			raw_spin_lock(&rq->lock);
			ttwu_do_activate(rq, p, 0);
			raw_spin_unlock(&rq->lock);
		}

### Use Cases

#### sleep_on()

很久很久以前, 大家使用这种方法来等待某个条件

	while (we_have_to_wait)
		sleep_on(&some_wait_queue);

#### wait_event()

### Schedule Timing

## Load Balance

## CFS
