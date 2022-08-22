---
title: "Linux内核教程(1) - 道路千万条，调试最重要"
date: 2022-08-23T00:14:58+08:00
draft: false
---

## 从信号量说起

大家可能都学过操作系统，在操作系统课上，在进程同步互斥中，图灵奖获得者Dijkstra的信号量Semphone。

Linux中当然也提供了semphone的实现，用做最普通的睡眠锁。所谓睡眠锁，意思是如果有一个任务试图去获取一个被占用的信号量时，会被推到等待队列中，然后让其睡眠。这样CPU资源就可以用来处理别的事情，实现资源的合理利用。这与一直等待的自旋锁形成鲜明的对比。当占有信号量的任务运行结束后，会唤醒队列里等待的任务，这个信号量也会被唤醒的任务占有。
针对于P和V两种原语的Linux实现是down和up两个操作。还有支持被中断的down_interruptible，可被杀的down_killable，不等待的down_trylock，带超时的down_timeout，考虑得非常周到。
不仅如此，信号量也是支持读/写信号量分离的。
一切看起来很美好，不是么？

我们看看semphone的作者在semphone.c的开头是如何写的：
```c
2  /*
3   * Copyright (c) 2008 Intel Corporation
4   * Author: Matthew Wilcox <willy@linux.intel.com>
5   *
6   * This file implements counting semaphores.
7   * A counting semaphore may be acquired 'n' times before sleeping.
8   * See mutex.c for single-acquisition sleeping locks which enforce
9   * rules which allow code to be debugged more easily.
10   */
```

对于懒得看英文的同学，我简单翻译一下，如果只是获取一次的锁，建议改用mutex.h，这样会使调试更容易。

在Linux kernel中，为了方便调试，基本上每种机制都有自己的调试宏，以CONFIG_DEBUG_*开头。下面是我随便搜的几个：
![config_debug](https://upload-images.jianshu.io/upload_images/1638145-fce32a60474c347e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

比如自旋锁，就有CONFIG_DEBUG_SPINLOCK，打开之后，会增加追踪如下例：
```c
3492  static inline void
3493  prepare_lock_switch(struct rq *rq, struct task_struct *next, struct rq_flags *rf)
3494  {
3495  	/*
3496  	 * Since the runqueue lock will be released by the next
3497  	 * task (which is an invalid locking op but in the case
3498  	 * of the scheduler it's an obvious special-case), so we
3499  	 * do an early lockdep release here:
3500  	 */
3501  	rq_unpin_lock(rq, rf);
3502  	spin_release(&rq->lock.dep_map, _THIS_IP_);
3503  #ifdef CONFIG_DEBUG_SPINLOCK
3504  	/* this is a valid case when another task releases the spinlock */
3505  	rq->lock.owner = next;
3506  #endif
3507  }
```

很不幸，semaphone不支持自动调试宏，连受限的也做不到。
所以，信号量最好的场景是特别复杂的场景，比如跨内核空间和用户空间的复杂交互类的，反正调试也不靠这个。
而对于内核代码中正常的使用，应该使用semaphone的受限版本mutex。
mutex是通过怎样的自律来获取自由的呢：
- 任何时间，只能有一个任务持有mutex
- 因为只有一个，所以加锁者必须负责给mutex解锁
- 因为要负责解锁，所以持有mutex的进程不得退出
- mutex不得用于中断处理程序，也包括下半部。不了解中断和下半部原理的我们后面会介绍
- mutex不能复制
- mutex不能手动初始化
- mutex只能初始化一次

加了这些自律之后，我们终于可以为mutex写一些调试用的功能了。不像自旋锁只是在处理上加了几条语句，mutex专门设计了调试专用函数来做这些事情：
```c
17  extern void debug_mutex_lock_common(struct mutex *lock,
18  				    struct mutex_waiter *waiter);
19  extern void debug_mutex_wake_waiter(struct mutex *lock,
20  				    struct mutex_waiter *waiter);
21  extern void debug_mutex_free_waiter(struct mutex_waiter *waiter);
22  extern void debug_mutex_add_waiter(struct mutex *lock,
23  				   struct mutex_waiter *waiter,
24  				   struct task_struct *task);
25  extern void mutex_remove_waiter(struct mutex *lock, struct mutex_waiter *waiter,
26  				struct task_struct *task);
27  extern void debug_mutex_unlock(struct mutex *lock);
28  extern void debug_mutex_init(struct mutex *lock, const char *name,
29  			     struct lock_class_key *key);
```

注：本文中的代码取自kernel 5.9.10版。

从信号量的例子我们就可以看到可调试性在内核中的重要性。

同样，有很多在内核开发中被重点强调的内容，其重要原因也是因为难以调试，比如栈溢出。
因为不像用户空间的应用程序容易退出，内核本身是一直长期运行的，这就导致内核不得不面对内存严重碎片化的情况，想要分配连续页的内存会越来越困难。而且用作栈的内存也没有办法换出到辅助存储中去，所以尽管栈溢出调试困难，也只能分配4k大小的栈。所以就要求开发者以自律享受自由，尽量避免在栈上分量大的对象。一旦栈溢出了，产生的结果是难以预测的。

所以，我们把内核的可调试性当作第一要务来强调。后面我们会调用各种手段来对内核进行调试，包括打日志，调试文件系统，perf和ftrace等工具，甚至SystemTap和eBPF这样的自动生成内核模块的脚本工具等，以及通过模拟器进行调试等各种手段。

## 编译内核

讲了调试的重要性之后，我们身体力行，首先讨论如何编译内核，如何在模拟器上跑起内核。
目前Linux的两个最主要的应用场景：一是跑在电脑上，主要场景是给自己的电脑更换内核；另一个是跑在嵌入式设备上，比如手机等。

### 下载内核

内核源代码地址可以在kernel.org上下载，比如我写此文时最新的稳定版是5.10.1，我们就可以下载这个包：
```
wget -c https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.10.1.tar.xz
```

如果要下载源码树的话，可以去clone kernel主线的代码库：
```
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

在运行Linux的电脑上，我们可以通过包管理系统获取到当前使用的系统内核的源代码。

比如在Ubuntu上，可以通过apt install linux-source来安装源代码：
```
root@iZ8vb39159pi4fttv8aaoyZ:/boot# apt search linux-source
Sorting... Done
Full Text Search... Done
linux-source/focal-updates,focal-updates,focal-security,focal-security,now 5.4.0.58.61 all [installed]
  Linux kernel source with Ubuntu patches

linux-source-5.4.0/focal-updates,focal-updates,focal-security,focal-security,now 5.4.0-58.64 all [installed,automatic]
  Linux kernel source for version 5.4.0 with Ubuntu patches
```

源代码会下载到/usr/src目录下。

### 编译Ubuntu 20.04的内核

下载之后，我们将其解压之后，就可以进行编译了。
安装gcc, flex, bison, bc之类的常规操作就不多说了，有遇到问题的请在评论区里提问。

编译之前需要对内核进行一些配置，比如调试信息等。这些配置会写在一个config文件中。
以Ubuntu 20.04系统为例，在/boot目录下可以看到一些config开头的文件，比如config-5.4.0-58-generic，这些就是我们使用的Ubuntu系统的config文件。

这些配置项，以CONFIG_* = y这样的形式来表示这个配置被支持。而如果不支持的话，则把这个配置项用“#”注释掉，并将=y改为is not set以利于理解。

比如我们上面讨论的锁的调试信息，在config文件中的描述如下：
```
#
# Lock Debugging (spinlocks, mutexes, etc...)
#
CONFIG_LOCK_DEBUGGING_SUPPORT=y
# CONFIG_PROVE_LOCKING is not set
# CONFIG_LOCK_STAT is not set
# CONFIG_DEBUG_RT_MUTEXES is not set
# CONFIG_DEBUG_SPINLOCK is not set
# CONFIG_DEBUG_MUTEXES is not set
# CONFIG_DEBUG_WW_MUTEX_SLOWPATH is not set
# CONFIG_DEBUG_RWSEMS is not set
# CONFIG_DEBUG_LOCK_ALLOC is not set
# CONFIG_DEBUG_ATOMIC_SLEEP is not set
# CONFIG_DEBUG_LOCKING_API_SELFTESTS is not set
# CONFIG_LOCK_TORTURE_TEST is not set
# CONFIG_WW_MUTEX_SELFTEST is not set
# end of Lock Debugging (spinlocks, mutexes, etc...)
```

我们以Ubuntu 20.04的源码为例，看看如何去编译内核。
进入/usr/src/linux-source-5.4.0目录，我们会看到linux-source-5.4.0.tar.bz2，将其解压。
进入解压后的目录，将/boot/config-5.4.0-58-generic文件复制过来。
然后执行
```
make ./config-5.4.0-58-generic
```
成功后，运行make menuconfig，在字符图形界面下可以进行一些手动的配置：
![kernel_config.png](https://upload-images.jianshu.io/upload_images/1638145-b665a856ca427654.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
配置好之后，保存到.config中，最后执行make -j4来进行编译。j后面是编译开启的线程数。

编译成功后，会看到类似于下面的输出：
```
Setup is 16380 bytes (padded to 16384 bytes).
System is 8697 kB
CRC 1a8c27e4
Kernel: arch/x86/boot/bzImage is ready  (#1)
```
编好的kernel在arch/x86/boot/bzImage。

如果我们是想在模拟器上运行内核的话，没有Ubuntu给我准备config。这也不怕，我们可以用x86_64的默认config，在其基础上进行修改。首先我们需要设置下ARCH变量为x86_64，这样不用写路径，make就知道去哪里找x86_64_defconfig
```
export ARCH=x86_64
make x86_64_defconfig
```

然后make menuconfig和make不变。

成功后输出如下：
```
Kernel: arch/x86/boot/bzImage is ready  (#2)
```

### 编译ARM64内核

x86_64的搞定了，换成别的架构就是照方抓药了。只不过需要装交叉编译的工具链。
在Ubuntu上，我们可以通过apt install gcc-aarch64-linux-gnu来安装支持ARM64的工具链。
安装好之后，我们配置下CROSS_COMPILE环境变量：
```
export CROSS_COMPILE=aarch64-linux-gnu-
```
针对于arm64，只有一个defconfig，设好ARCH之后就可以自动找到了：
```
export ARCH=arm64
```

ARCH后面的名字以arch下的子目录名为准，目前kernel支持的架构如下：
- arc  
- arm64  
- csky   
- hexagon  
- m68k        
- mips   
- nios2     
- parisc   
- riscv  
- sh     
- um   
- x86_64
- alpha    
- arm  
- c6x    
- h8300  
- ia64     
- microblaze  
- nds32  
- openrisc  
- powerpc  
- s390   
- sparc  
- x86  
- xtensa

我们执行make defconfig

```
make defconfig
```
然后运行make -j8之类就可以了。

### init程序

但是这样编出来的内核，真的只是一个内核，没有任何shell之类的可以用。内核启动的最后，是要启动一个init程序的，当然我们也可以手写一个。但是为了能有个shell，我们选择用busybox的init. 

我们去busybox.net去下载源码：
```
wget -c https://busybox.net/downloads/busybox-1.32.0.tar.bz2
```
刚才编译内核时已经设置好ARCH和CROSS_COMPILE了，正好busybox也能用到。
busybox没那么多defconfig，上来就make menuconfig就好。
![busybox.png](https://upload-images.jianshu.io/upload_images/1638145-4c6a7538b66c5125.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们只需要一个busybox程序，所以选择Build static binary。
退出保存之后，执行make -j4去编译。
最后，执行make install，会安装到_install目录下。

准备好了之后，我们需要给busybox的init准备一个配置文件，一般是/etc/init.d/inittab。这时候别说inittab了，我们连目录还没建呢。

第一步：在_install目录下创建etc,dev,mnt和etc/init.d/目录：
```
mkdir etc
mkdir dev
mkdir mnt
mkdir -p etc/init.d/
```

mkdir加-p参数的意思是如果父目录没有创建，则创建之。

第二步：创建inittab文件。
这个我们哪会写，看busybox给我们的例子：
```
::sysinit:/etc/init.d/rcS
# /bin/sh invocations on selected ttys
#
# Note below that we prefix the shell commands with a "-" to indicate to the
# shell that it is supposed to be a login shell.  Normally this is handled by
# login, but since we are bypassing login in this case, BusyBox lets you do
# this yourself...
#
# Start an "askfirst" shell on the console (whatever that may be)
::askfirst:-/bin/sh
# Start an "askfirst" shell on /dev/tty2-4
tty2::askfirst:-/bin/sh
tty3::askfirst:-/bin/sh
tty4::askfirst:-/bin/sh

# /sbin/getty invocations for selected ttys
tty4::respawn:/sbin/getty 38400 tty5
tty5::respawn:/sbin/getty 38400 tty6

# Stuff to do when restarting the init process
::restart:/sbin/init

# Stuff to do before rebooting
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a
```

我们把注释删一删，tty也用不了这么多，精简一下：
```
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh
::restart:/sbin/init
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a
```

另外，我们不想用::respawn:-/sbin/getty或者login之类的登陆界面，直接进入系统，所以加一条直接调shell: ::respawn:-/bin/sh。
这个参考自busybox的examples/bootfloppy/etc下面的inittab

最后写出来如下：
```
::sysinit:/etc/init.d/rcS
::askfirst:-/bin/sh
::respawn:-/bin/sh
::restart:/sbin/init
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a
```

写完之后，我们欠Busybox一个启动脚本/etc/init.d/rcS。

第三步：在etc/init.d目录下创建rcS文件，如下：
```
mkdir -p /proc
mkdir -p /tmp
mkdir -p /sys
mkdir -p /mnt
/bin/mount -a
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

/proc是内核向进程发送消息的机制。比如cat /proc/cpuinfo可以查看cpu运行信息，而cat /proc/meminfo是内存信息。
/sys与/proc类似，也是内核用于展示信息的虚拟文件系统，于2.5版引入，主要展示设备树。
/tmp是临时目录
/mnt是挂载点
/dev/pts是通过ssh等远程登陆时创建的控制台设备文件

mdev是busybox提供的管理热插拔的程序。
```
BusyBox v1.32.0 (2020-12-15 16:24:44 CST) multi-call binary.

Usage: mdev [-s] | [-df]

mdev -s is to be run during boot to scan /sys and populate /dev.
mdev -d[f]: daemon, listen on netlink.
	-f: stay in foreground.

Bare mdev is a kernel hotplug helper. To activate it:
	echo /sbin/mdev >/proc/sys/kernel/hotplug
```
如上面说明所示，-s用于启动时扫描，激活命令我们也照抄。

rcS写好了之后需要通过chmod +x rcS赋给可执行权限。

有同学问了，mount -a是挂载啥的？这是按照/etc/fstab来mount所有里面写的文件系统的，我们马上就写一个fstab。
参照busybox-1.32.0/examples/bootfloppy/etc/init.d/rcS，mount -a调用fstab也是busybox的传统操作。

第四步, 创建fstab文件
内容如下：
```
proc /proc proc defaults 0 0
tmpfs /tmp tmpfs defaults 0 0
sysfs /sys sysfs defaults 0 0
tmpfs /dev tmpfs defaults 0 0
debugfs /sys/kernel/debug debugfs defaults 0 0
```

到此为止，欠busybox init的连环债算是还清了，下面我们就以此文件系统去编译内核。

### 在qemu中运行

有了busybox的init程序，我们重新编译下内核，在menuconfig中将刚才的_install目录设置进去，在General配置的Init RAM filesystem中：
![initramfs](https://upload-images.jianshu.io/upload_images/1638145-4262e190bf23982a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将我们刚才准备好的_install目录的路径设置进去就好。

配置好保存之后，make -j4开始编译。

经过一段欢快的编译，生成Image:
```
...
  LD      vmlinux.o
  MODPOST vmlinux.symvers
  MODINFO modules.builtin.modinfo
  GEN     modules.builtin
  LD      .tmp_vmlinux.kallsyms1
  KSYMS   .tmp_vmlinux.kallsyms1.S
  AS      .tmp_vmlinux.kallsyms1.S
  LD      .tmp_vmlinux.kallsyms2
  KSYMS   .tmp_vmlinux.kallsyms2.S
  AS      .tmp_vmlinux.kallsyms2.S
  LD      vmlinux
  SORTTAB vmlinux
  SYSMAP  System.map
  MODPOST Module.symvers
  OBJCOPY arch/arm64/boot/Image
  GZIP    arch/arm64/boot/Image.gz
```

下面我们调用qemu来运行这个内核，qemu可以通过apt来安装。我们模拟4核A72，16G内存：
```
qemu-system-aarch64 -machine virt -cpu cortex-a72 -machine type=virt -nographic -m 16384 -smp 4 -kernel arch/arm64/boot/Image --append "rdinit=/linuxrc console=ttyAMA0"
```

然后我们就可以登陆进我们的aarch64的Linux啦，我们可以uname看看，是不是我们编的5.10.1:
```
/ # uname -a
Linux (none) 5.10.1 #4 SMP PREEMPT Wed Dec 16 18:19:47 CST 2020 aarch64 GNU/Linux
```

我们再看看cpuinfo:
```
/ # cat /proc/cpuinfo
processor	: 0
BogoMIPS	: 125.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3

processor	: 1
BogoMIPS	: 125.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3

processor	: 2
BogoMIPS	: 125.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3

processor	: 3
BogoMIPS	: 125.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd08
CPU revision	: 3
```

最后的秘技是如何退出qemu，按Ctrl-a x，就可以退出了，显示：
```
/ # QEMU: Terminated
```

恭喜，一个可玩的内核已经可以工作啦。

## 懂源码是根本

Linus反对在内核里加入调试器也不是没有道理，调试器只是手段，我们也不能舍本逐末，有了方便的调试手段就不去钻研原理和源码了。
我们希望在解剖kernel的时候能让大家有更丰富的视角，但是最近我们的目标还是理解内核的逻辑和代码。
请大家跟我一起沉下心来，我们一步一步开始探索之旅。

