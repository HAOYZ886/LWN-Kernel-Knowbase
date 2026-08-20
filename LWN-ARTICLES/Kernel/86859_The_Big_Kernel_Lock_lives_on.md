---
title: The Big Kernel Lock lives on
url: https://lwn.net/Articles/86859/
date: "May 26, 2004"
category: Big kernel lock; ioctl; lock kernel
author: 
---

It was recently [noted](<https://lwn.net/Articles/86860/>) that `ioctl()` system calls are still executed with the Big Kernel Lock (BKL) held. A suggestion was made that drivers which can implement `ioctl()` without the BKL held should be specially flagged as a way of increasing parallelism. That suggestion looks like it will not get very far. But it did pique your editor's interest in current use of the BKL. Besides, there hasn't been a whole lot else going on this week. 

The BKL is an artifact from when the Linux kernel first supported multiprocessor systems. Making the kernel safe for concurrent access from multiple CPUs has been a multi-year task; it is not a job that could have been done all at once at the beginning. So Linux 2.0 supported SMP systems by way of the BKL, which only allowed one processor to be running kernel code at any given time. The BKL is essentially a spinlock, but with a couple of interesting properties: 

  * The BKL can be taken recursively; the kernel remembers how many times a given thread has called `lock_kernel()` and does the right thing. Normal spinlocks are rather less forgiving. 

  * Code holding the BKL can sleep. The lock is released while the given thread sleeps, and reacquired upon awakening. 

The BKL made SMP Linux possible, but it didn't scale very well. Its overhead could be felt even with two processors, and it made running on anything larger problematic. So the kernel developers have been breaking the BKL into finer-grained locks ever since. Thus, for example, the block I/O subsystem went from the BKL to its own lock (`io_request_lock`) in 2.2, and from that to individual queue locks in 2.6. The kernel now has thousands of locks, and some people had assumed that the BKL would be gone by 2.6. 

As it turns out, there are still over 500 `lock_kernel()` calls in the 2.6.6 kernel. For the curious, here are some of the places which still rely on this old, system-wide lock: 

  * The core kernel retains a few calls. The implementation of the `reboot()` system call is one of them; this is, of course, not one of the more performance-sensitive parts of the kernel. The boot-time early initialization process is also run with the BKL held. The `sysctl()` system call is run under the BKL; interestingly, while much of `/proc` is also implemented under the BKL, it appears that reads and writes to `/proc/sys` do not run with the BKL held. 

  * Many older filesystems (UFS, coda, HPFS, FAT, NCP, SMB, Minix, etc.) make heavy use of the BKL for serialization. The UnixWare "Boot File System" implementation has several calls; somehow, they seem unlikely to be fixed anytime soon. There are also `lock_kernel()` calls in NFS, UDF, isofs, the reiserfs journaling code, autofs, and some others. The ext2 filesystem uses the BKL to protect modifications to the superblock; ext3, instead, had all of its `lock_kernel()` calls purged during the 2.5 development process. 

  * The `rpciod` kernel thread spends its entire life with the BKL held. 

  * Core dumps are created with the BKL held. 

  * Block and character devices have their `open()` methods called under the BKL. Block `release()` methods are also called this way, but that is not true for char drivers. The default `llseek()` method runs under the BKL, but, if a driver or filesystem provides its own `llseek()` method, that method will not be called with the BKL held. The `fasync()` method is always called under the BKL. As noted at the beginning, `ioctl()` methods are called with the lock held; additionally, the ugly code which does 32-bit emulation on 64-bit systems needs the BKL. 

  * The file locking code still requires the BKL. 

  * Almost 10% of the `lock_kernel()` calls can be found in the (old, deprecated) OSS sound code. The ALSA code has no BKL calls, with one exception: the implementation of its `/proc` files. 

  * Most of the architectures retain some calls in the arch-specific code. The `ptrace()` system call is one common place for these calls. i386 also uses the BKL to protect `llseek()` calls on the CPUID and MSR pseudo-devices. uClinux performs `execve()` calls under the BKL. 

  * Almost all of the remaining BKL calls are to be found in device drivers. The TTY subsystem still has quite a few of them, as does USB. Many of these calls are protecting `llseek()` implementations. Quite a few of the rest are for the creation of special-purpose kernel threads: the `daemonize()` function needs to be called with the BKL held. Those calls can, presumably, go away as the driver code is (slowly) migrated over to the new kthread calls. 

Given how poorly the BKL is viewed, it may be surprising that so many places in the kernel still use it. The simple fact is that, with regard to the BKL, all of the low-hanging fruit has long since been taken. For most of the remaining calls, removing the BKL is not worth the trouble and code churn. So, while removal of the remaining calls over the 2.7 development series looks entirely possible, it would not be surprising if that does not happen.
