---
layout: post
title: RISC-V Atomic LR/SC vs AMO
author: "Fei Wu"
header-mask: 0.4
tags:
  - RISC-V
  - Performance
  - Atomic
---

在sohpgo sg2038 (64core) 上面测试vm-scalability的时候发现一个有趣的现象，64个cpu上的%sys长期保持在100%，cpu都耗在了内核的原子操作上面。

# 测试用例

测试使用的命令行如下，usemem分配了1GB空间并进行随机写入操作。

```
 Performance counter stats for './usemem --runtime 90 -t 64 --prealloc --random 1055820736':

       11134377.19 msec task-clock                       #   63.226 CPUs utilized
             22776      context-switches                 #    2.046 /sec
              1621      cpu-migrations                   #    0.146 /sec
           1626092      page-faults                      #  146.042 /sec
    22268290761022      cycles                           #    2.000 GHz
      104991871953      instructions                     #    0.00  insn per cycle
       25806654302      branches                         #    2.318 M/sec
         412629425      branch-misses                    #    1.60% of all branches

     176.104518759 seconds time elapsed

       1.114411000 seconds user
   11063.446212000 seconds sys
```

# 性能分析

逻辑上讲，1GB空间只需要256k次page fault就够了，上面1626092次page faults显然是有其他原因导致而增多的，分析后是由于numa balancing导致，所以测试的时候可以先配置如下, 这样page faults的次数就会接近256K。

* 关闭numa balancing，/proc/sys/kernel/numa_balancing
* 关闭thp, /sys/kernel/mm/transparent_hugepage/enabled

使用perf抓到的调用栈基本都在拿锁的地方

```
-   97.51%    97.51%  usemem   [kernel.vmlinux]  [k] down_read_trylock                                                                                                                                           ▒
     do_access                                                                                                                                                                                                   ▒
     ret_from_exception                                                                                                                                                                                          ▒
     do_page_fault                                                                                                                                                                                               ▒
     handle_page_fault                                                                                                                                                                                           ▒
     lock_vma_under_rcu                                                                                                                                                                                          ▒
     down_read_trylock                                                                                                                                                                                           ▒
```

具体执行的指令就是在lr/sc里面

```
       │    raw_atomic64_cmpxchg_acquire():                                                                                                                                                                         0.22 │40:   lr.d   a2,(s1)
 98.30 │    → bne    a2,a5,7ff959af
  0.22 │      sc.d   a1,a3,(s1)
  1.14 │    ↑ bnez   a1,40                                                                                                                                                                                               │      fence  r,rw                                                                                                                                                                                         
```

# 构建用例

根据以上分析，在冲突严重的时候，lr/sc造成了性能的极端情况，我们设计一个小的测试用例方便分析。

这里使用了https://github.com/michaeljclark/riscv-atomics.git里面的接口，其实并不是完全必要，可以使用gcc等的builtin函数。

```
#define _GNU_SOURCE
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include <sched.h>
#include <stdlib.h>

#define ATOMIC_ASM 1
#include "stdatomic.h"

#define MAX_CPU 64

struct thread_data {
	int cpu;         /* pin to this cpu */
	int loop;        /* ops count */
	long *shared;    /* atomic add to this */
};

pthread_barrier_t start_barrier;

int use_lr_sc = 1;

int pin_myself(int cpu) {
	cpu_set_t cpuset;
	CPU_ZERO(&cpuset);
	CPU_SET(cpu, &cpuset);

	pthread_t current_thread = pthread_self();	 
	return pthread_setaffinity_np(current_thread, sizeof(cpu_set_t), &cpuset);
}

void *thread(void *ptr)
{
	struct thread_data *td = ptr;
	int loop = td->loop;
	long i;

	pin_myself(td->cpu);

	pthread_barrier_wait(&start_barrier);

	if (use_lr_sc == 1) {
		for (i = 0; i < loop; ++i) {
			long tmp = atomic_load_explicit(td->shared, memory_order_relaxed);
			while (1) {
				long r = __atomic_cmpxchg_acq_rel(td->shared, &tmp, tmp + 1);

				if (r == tmp) {
					break;
				} else {
					tmp = r;
				}
			}
		}
	} else if (use_lr_sc == 0) {
		for (i = 0; i < loop; ++i) {
			atomic_fetch_add(td->shared, 1);
		}
	}

	return NULL;
}

int main(int argc, char *argv[])
{
	pthread_t pth[MAX_CPU];
	struct thread_data td[MAX_CPU];
	long value = 0;
	long total_cas_count = 0;
	int loop = 10000;

	int num_thr = MAX_CPU;

	if (argc > 1) {
		use_lr_sc = atoi(argv[1]);
	}

	if (argc > 2) {
		num_thr = atoi(argv[2]);
		if (num_thr > MAX_CPU) {
			printf("too many cpus\n");
			return 1;
		}
	}

	if (argc > 3) {
		loop = atoi(argv[3]);
	}

	printf("num_thr: %d, use_lr_sc: %d\n", num_thr, use_lr_sc);

	pthread_barrier_init(&start_barrier, NULL, num_thr);

	for (int i = 0; i < num_thr; ++i) {
		td[i].cpu = i % MAX_CPU;
		td[i].shared = &value;
		td[i].loop = loop;
		int ret = pthread_create(&pth[i], NULL, thread, (void *)&td[i]);
		if (ret != 0) {
			printf("pthread_create failed\n");
			return 1;
		}
	}

	for (int i = 0; i < num_thr; ++i) {
		pthread_join(pth[i], NULL);
	}

	pthread_barrier_destroy(&start_barrier);

	printf("final value: %ld, loop: %d\n", value, loop);
	return 0;
}

```

# 新用例分析

编译选择对性能会有影响，这里选择O0不优化

> $ gcc test-atomic.c -g -O0 -o atomic.O0

## LR/SC vs AMO

分别测试LR/SC和AMO的效果，可以看到在强竞争环境下AMO的效果会好非常多
* 性能差距在1000x级别
* LR/SC的性能结果方差较大，有时出现好几倍的变化. 可能跟系统的状态比如cache等有关系，后续可以看看是否有相应PMU来分析

```
$ sudo perf stat ./atomic.O0 1
num_thr: 64, use_lr_sc: 1
final value: 640000, loop: 10000

 Performance counter stats for './atomic.O0 1':

        2141505.16 msec task-clock                       #   55.357 CPUs utilized
              1389      context-switches                 #    0.649 /sec
                64      cpu-migrations                   #    0.030 /sec
               177      page-faults                      #    0.083 /sec
     4282984915631      cycles                           #    2.000 GHz
        6609772363      instructions                     #    0.00  insn per cycle
        2455506052      branches                         #    1.147 M/sec
          35676085      branch-misses                    #    1.45% of all branches

      38.685101551 seconds time elapsed

    2141.456224000 seconds user
       0.049997000 seconds sys


$ sudo perf stat ./atomic.O0 0
num_thr: 64, use_lr_sc: 0
final value: 640000, loop: 10000

 Performance counter stats for './atomic.O0 0':

           1364.05 msec task-clock                       #   24.749 CPUs utilized
               190      context-switches                 #  139.291 /sec
                64      cpu-migrations                   #   46.919 /sec
               177      page-faults                      #  129.760 /sec
        2722508418      cycles                           #    1.996 GHz
          68832229      instructions                     #    0.03  insn per cycle
           5228336      branches                         #    3.833 M/sec
            438094      branch-misses                    #    8.38% of all branches

       0.055116118 seconds time elapsed

       1.346023000 seconds user
       0.031547000 seconds sys
```

## 高并发的LR/SC

测试不同并发条件下LR/SC的表现

```
$ for i in 8 16 32 64; do sudo perf stat ./atomic.O0 1 $i; done
```

| 并发 | seconds user (perf stat) |
| ---- | ------------------------ |
| 8    | 0.979668000 |
| 16   | 40.213113000 |
| 32   | 342.519529000 |
| 64   | 2007.073680000 |

## 低并发的LR/SC

这里改大了循环次数

```
$ for i in 1 2 4 8; do sudo perf stat ./atomic.O0 1 $i 2000000; done
```

| 并发 | seconds user (lr/sc) | amo |
| ---- | -------------------- | --- |
| 1    | 0.142973503 | 0.117086000 |
| 2    | 0.510705000 | 0.263231000 |
| 4    | 2.302510000 | 0.718847000 |
| 8    | 200.607623000 | 4.448020000 |

# 后续

还有很多事情可以做，比如:
* 在x86上测试，和sophgo进行比较
* 查看是否有对应PMU可以观察, 比如lr/sc性能不稳定的来源
* 软件上是不是可以通过插入其他指令来避免lr/sc带来的竞争
* 硬件上是不是有办法动态调节lr/sc的竞争及时退让, 预计这方面在x86/arm等系统上有经验
