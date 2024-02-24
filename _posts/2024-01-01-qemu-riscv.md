---
layout: post
title: QEMU做RISC-V开发
subtitle: "工欲善其事，必先利其器"
author: "Fei Wu"
header-mask: 0.4
tags:
  - Qemu
  - Linux
  - RISC-V
  - Debug
---

* content
{:toc}

很多情况QEMU都是系统开发的好选择，甚至时最好的选择。 对于现在的RISC-V开发来说更是如此，各种新的指令集非指令集扩展不停的发布，QEMU是支持的最早基本也是最好的平台。

# 下载RISC-V镜像

Ubuntu较早支持RISC-V，同时镜像发布比较正规也很容易搜到

> https://cdimage.ubuntu.com/ubuntu/releases/23.10/release/

当然我们也可以选择debian，可以从这里下载

> https://people.debian.org/~gio/dqib/

我一般并不太关注哪个具体发行版，选择一个好用的就行，特别地，我会选择preinstalled disk image，做到拿来即用。

# 启动QEMU

正常情况会使用bootloader比如这里的uboot的启动系统。

~~~
qemu-system-riscv64 -machine virt -cpu rv64 -m 1G -nographic \
    -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd \
    -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 \
    -bios /usr/lib/riscv64-linux-gnu/opensbi/generic/fw_jump.elf \
    -kernel /usr/lib/u-boot/qemu-riscv64_smode/uboot.elf
~~~

如果是用作kernel开发的话，在qemu的命令行直接传入kernel image会比较方便。特别地，如果kernel已经builtin了启动qemu所需的驱动，只需要修改这一个地方就可以调试kernel了。

~~~
qemu-system-riscv64 -machine virt -cpu rv64 -m 1G -nographic \
    -device virtio-blk-device,drive=hd -drive file=image.qcow2,if=none,id=hd \
    -device virtio-net-device,netdev=net -netdev user,id=net,hostfwd=tcp::2222-:22 \
    -bios /usr/lib/riscv64-linux-gnu/opensbi/generic/fw_jump.elf \
    -kernel $kernel_file \
    -initrd $initrd_file \
    -append "root=LABEL=cloudimg-rootfs ro earlycon"
~~~

看到ubuntu login, riscv环境配置就完成了，我们可以用它来调试内核，用户程序，以及固件等。

~~~
[  OK  ] Started User Login Management.
[  OK  ] Started Unattended Upgrades Shutdown.
[  OK  ] Started CUPS Scheduler.
[  OK  ] Started Modem Manager.
[  OK  ] Started Disk Manager.

Ubuntu 22.04.2 LTS ubuntu ttyS0

ubuntu login:
~~~

# 简单内核调试 

假设我们想要调试kernel的初始化代码，我们在qemu的命令行上加上一下参数, 此时qemu会等待gdb的连接

> -s -S

然后用gdb-multi连上并进行调试

~~~
$ gdb-multiarch vmlinux
(gdb) set arch riscv:rv64
The target architecture is set to "riscv:rv64".
(gdb) target remote :1234
Remote debugging using :1234
0x0000000000001000 in ?? ()
(gdb) b start_kernel
Breakpoint 1 at 0xffffffff80c009fa (2 locations)
(gdb) c
Continuing.
[Switching to Thread 1.3]

Thread 3 hit Breakpoint 1, 0xffffffff80c009fa in start_kernel ()
(gdb) b vfs_read
Breakpoint 2 at 0xffffffff802b4be4: vfs_read. (2 locations)
(gdb) c
Continuing.

Thread 3 hit Breakpoint 1, start_kernel () at ../init/main.c:941
941     {
(gdb) bt
#0  start_kernel () at ../init/main.c:941
#1  0xffffffff80001160 in _start_kernel () at ../arch/riscv/kernel/head.S:326
Backtrace stopped: frame did not save the PC
(gdb) p ((struct task_struct *)$tp)->comm
$1 = "swapper\000\000\000\000\000\000\000\000"
(gdb) p ((struct task_struct *)$tp)->pid
$2 = 0
~~~
