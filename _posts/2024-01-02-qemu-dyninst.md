---
layout: post
title: QEMU统计动态指令
subtitle: "工欲善其事，必先利其器"
author: "Fei Wu"
header-mask: 0.4
tags:
  - Qemu
  - Debug
  - Performance
  - RISC-V
---

* content
{:toc}

qemu tcg使用二进制翻译执行代码，用来统计动态指令再合适不过，甚至可以获取比真实硬件更详细的信息。理论上在guest内部通过perf tools来获取动态指令数也是可行的，不过至少现在的qemu riscv并不支持。这里通过使用qemu的插件libinsn来统计动态指令，该插件在系统模式的qemu可用，同样也能应用于用户模式的qemu。

# 使用方法

在qemu命令行加入如下操作即可使能libinsn

> -plugin $QEMU_SRC/build/tests/plugin/libinsn.so,inline=on -d plugin

qemu提供了2种模式来使用libinsn

* inline=on，也就是inline模式，开销较小，只统计整个guest的执行指令条数
* inline=off，也就是percpu模式，开销较大，会统计每个vcpu的执行指令条数

# 实现简介

我们这里仅以inline模式为例，比如有如下guest指令

```
li t1, 0x41
li t2, 0x42
rdtime t3
```

翻译为host的指令流为：

```
0x00007fffe800010b <+222>:   lea    0xffc7f46(%rip),%rbx        # 0x7ffff7fc8058 <inline_insn_count>
0x00007fffe8000112 <+229>:   mov    (%rbx),%r12
0x00007fffe8000115 <+232>:   inc    %r12
0x00007fffe8000118 <+235>:   mov    %r12,(%rbx)
0x00007fffe800011b <+238>:   movq   $0x41,0x30(%rbp)
0x00007fffe8000123 <+246>:   mov    (%rbx),%r12
0x00007fffe8000126 <+249>:   inc    %r12
0x00007fffe8000129 <+252>:   mov    %r12,(%rbx)
0x00007fffe800012c <+255>:   movq   $0x42,0x38(%rbp)
0x00007fffe8000134 <+263>:   mov    (%rbx),%r12
0x00007fffe8000137 <+266>:   inc    %r12
0x00007fffe800013a <+269>:   mov    %r12,(%rbx)
0x00007fffe800013d <+272>:   mov    $0xc01,%esi
0x00007fffe8000142 <+277>:   mov    %rbp,%rdi
0x00007fffe8000145 <+280>:   callq  *0x25(%rip)        # 0x7fffe8000170 <code_gen_buffer+323>
0x00007fffe800014b <+286>:   mov    %rax,0xe0(%rbp)
0x00007fffe8000152 <+293>:   movq   $0x100bc,0x1330(%rbp)
0x00007fffe800015d <+304>:   jmpq   0x7fffe8000016
0x00007fffe8000162 <+309>:   lea    -0x126(%rip),%rax        # 0x7fffe8000043 <code_gen_buffer+22>
0x00007fffe8000169 <+316>:   jmpq   0x7fffe8000018
0x00007fffe800016e <+321>:   nop
0x00007fffe800016f <+322>:   nop
0x00007fffe8000170 <+323>:   movabs 0x5555555a44,%al
```

可以看到在每条指令之前，都会将inline_insn_count计数加一

```
0x00007fffe8000112 <+229>:   mov    (%rbx),%r12
0x00007fffe8000115 <+232>:   inc    %r12
0x00007fffe8000118 <+235>:   mov    %r12,(%rbx)
```

大的逻辑上比较简单。

# 简单验证

我们写个确定的测试程序, 最后ecall会执行exit系统调用并退出。

```
.global _start

.text
_start:
    li t1, 0x41
    fmul.d  fa5,fa5,fa3
    fdiv.d  fa5,fa5,fa4
    li t2, 0x42
    rdtime t3
    li t4, 0x44
    li a7, 93
    ecall
```

并编译成可执行程序

```
$ riscv64-linux-gnu-gcc -c fixed.s -o fixed.o
$ riscv64-linux-gnu-ld fixed.o -o fixed
```

我们使用user mode qemu执行，期望执行返回8，真实结果返回8。

```
$ qemu-riscv64 -plugin ./libinsn.so,inline=on -d plugin ./fixed
insns: 8
```

# 工具设计

工具的需求需要实时输出guest的指令条数，比如间隔一秒钟输出一次guest的动态指令数。因为libinsn已经实现相应的逻辑，我们只需要把对应信息拿到并进行计算即可，这个可以通过gdb来获取，也可以通过ebpf工具获取。

## 获取count地址

```
#!/bin/bash

if (( "$#" < 1 )); then
	echo "usage: $0 pid"
	exit 1
fi

pid=$1

qemu_path=$(readlink /proc/$pid/exe)
libinsn_path=$(lsof -p $pid 2>/dev/null | grep libinsn.so | tail -1 | awk '{print $NF}')
plugin_dir=$(dirname "$libinsn_path")

function is_inline()
{
	cmd=$(ps -p $pid -o command)
	if [[ $cmd =~ "inline=false" ]]; then
		echo "false"
	elif [[ $cmd =~ "inline" ]]; then
		echo "true"
	else
		echo "false"
	fi
}

# return addr global @var_addr
function get_var_addr()
{
	logfile=$(mktemp insn.XXX)
	sudo gdb -p $pid -batch-silent \
	  -ex "set solib-search-path $plugin_dir" \
	  -ex "set logging file $logfile" \
	  -ex "set logging on" \
	  -ex "p &$1" \
	  -ex "set logging enabled off" \
	  -ex quit 2>/dev/null

	var_addr=$(tail -1 $logfile | awk '{print $(NF-1);}')
	rm $logfile
}


inline=$(is_inline)
if [[ $inline == "true" ]]; then
	get_var_addr inline_insn_count
	count_addr=$var_addr
	sudo bpftrace ./qemu_icount_inline.bt $qemu_path $count_addr
else
	get_var_addr "counts[0]->insn_count"
	count_addr=$var_addr
	sudo bpftrace ./qemu_icount_cpu.bt $qemu_path $count_addr
fi
```

## inline模式统计

```
#!/usr/bin/bpftrace

// $1 - qemu path
// $2 - &inline_insn_count

uprobe:$1:cpu_exec {
	if (@prev_ns == 0) {
		@prev_ns = nsecs;
		@prev_count = *$2;
	}
	$dur = nsecs - @prev_ns;
	if ($dur > 1000000000) {
		@prev_ns = nsecs;
		$count = *$2;
		printf("time: %ld, dur: %ld, count: %ld\n", nsecs, $dur, $count - @prev_count);
		@prev_count = $count;
	}
}

END {
	clear(@prev_ns);
	clear(@prev_count);
}
```

## percpu模式统计

```
#!/usr/bin/bpftrace

// $1 - qemu path
// $2 - &counts[0]->insn_count, 4 cpus at max

uprobe:$1:cpu_exec {
	if (@prev_ns == 0) {
		@prev_ns = nsecs;
		@prev_count_cpu0 = *$2;
		@prev_count_cpu1 = *($2 + 16);
		@prev_count_cpu2 = *($2 + 32);
		@prev_count_cpu3 = *($2 + 48);
	}
	$dur = nsecs - @prev_ns;
	if ($dur > 1000000000) {
		@prev_ns = nsecs;
		$count_cpu0 = *$2;
		$count_cpu1 = *($2 + 16);
		$count_cpu2 = *($2 + 32);
		$count_cpu3 = *($2 + 48);
		printf("time: %ld, dur: %ld, count: %12ld,%12ld,%12ld,%12ld\n",
			nsecs, $dur,
			$count_cpu0 - @prev_count_cpu0,
			$count_cpu1 - @prev_count_cpu1,
			$count_cpu2 - @prev_count_cpu2,
			$count_cpu3 - @prev_count_cpu3);
		@prev_count_cpu0 = $count_cpu0;
		@prev_count_cpu1 = $count_cpu1;
		@prev_count_cpu2 = $count_cpu2;
		@prev_count_cpu3 = $count_cpu3;
	}
}

END {
	clear(@prev_ns);
	clear(@prev_count_cpu0);
	clear(@prev_count_cpu1);
	clear(@prev_count_cpu2);
	clear(@prev_count_cpu3);
}
```

## 使用效果

在guest里面执行如下命令

```
ubuntu@ubuntu:~/unixbench/UnixBench/pgms$ ./syscall 10 getpid
COUNT|11737472|1|lps
```

在host上每隔一秒输出动态指令执行结果

```
time: 337205296796152, dur: 1003998237, count: 295237595
time: 337206300795879, dur: 1004000060, count: 294935750
time: 337207300796814, dur: 1000000552, count: 293587477
time: 337208304796267, dur: 1003999414, count: 294696957
time: 337209304796755, dur: 1000000742, count: 293304814
time: 337210304797185, dur: 1000000105, count: 292635625
time: 337211309080176, dur: 1004280776, count: 180943397
time: 337212333077018, dur: 1023996710, count: 287779
time: 337213439824785, dur: 1106747438, count: 413462
time: 337214448606455, dur: 1008781644, count: 527250
```
