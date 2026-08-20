---
title: "Merged for 2.6.24, part 2"
url: https://lwn.net/Articles/255327/
date: "October 24, 2007"
category: "Releases-2.6.24"
author: "By Jonathan Corbet October 24, 2007"
---

By **Jonathan Corbet**  
October 24, 2007

The 2.6.24 merge window has now closed; more than 7000 changesets were merged before 2.6.24-rc1 was released. The bulk of the new features for 2.6.24 were [described last week](<http://lwn.net/Articles/254559/>). Here's a summary of patches merged since then, starting with user-visible changes: 

  * There are new drivers for Marvell Libertas 8385/6 wireless chips, Freescale 3.0Gbps SATA controllers, Fujitsu laptops (LCD brightness in particular), and TI AR7 watchdog devices. 

  * Another set of old Open Sound System drivers has been removed from the kernel. 

  * The "uninitialized block groups" feature has been merged into the ext4 filesystem. UBG helps to speed filesystem checks by keeping track of which parts of a disk partition have never been used, and, thus, do not require checking. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

  * As was [discussed back in August](<http://lwn.net/Articles/247243/>), the binary `sysctl()` interface has been marked deprecated, and the code for many of the sysctl targets (much of which appears to not have worked for some time) has been removed. There is a new checker which looks for problematic sysctl definitions; according to Eric Biederman, ""As best as I can determine all of the several hundred errors spewed on boot up now are legitimate."" 

  * The semantics of the `CAP_SETPCAP` capability have been changed. In previous kernels, this capability gave a process the ability to bestow new capabilities upon another process; now, instead, it allows a process to set capabilities within its own "inherited" mask. 

  * Process CPU time accounting (via taskstats) has been augmented with information allowing CPU usage time to be scaled by CPU frequency. 

  * The Control Groups (formerly [process containers](<http://lwn.net/Articles/236038/>)) patch set has been merged. Control groups will allow the CFS group scheduling feature to be used; it will also be the control mechanism used for containers in general. 

  * Process ID namespaces have been added; this feature lets container implementations create a different view of the list of processes on the system for every container. 

  * The [kernel markers](<http://lwn.net/Articles/245671/>) patch set has been merged. 

  * The CIFS filesystem now has access control list (ACL) support. 

  * The old, unmaintained Fibre Channel support code has been removed. 

Changes visible to kernel developers include: 

  * The process of merging the i386 and x86_64 architectures continues, with many files having been merged by the time the window closed. This job is far from complete, though. For the curious, [this message from Ingo Molnar](<https://lwn.net/Articles/255681/>) talks a bit about what is going on there. ""The x86 architecture is the most common Linux architecture after all - and users care much more about having a working kernel than they care about cleanups and unifications.... This cannot be realistically finished in v2.6.24, without upsetting the codebase."" 

  * The paravirt_ops structure has been split into several smaller, more specialized operations vectors. These include `pv_init_ops` (boot-time operations), `pv_time_ops` (for time-related operations), `pv_cpu_ops` (privileged instructions), `pv_irq_ops` (interrupt handling), `pv_mmu_ops` (page table management), and a few others. 

  * There are some new bit operations which have been added: 

```
int test_and_set_bit_lock(unsigned long nr, unsigned long *addr);
            void clear_bit_unlock(unsigned long nr, unsigned long *addr);
            void __clear_bit_unlock(unsigned long nr, unsigned long *addr);
```

These operations are intended to be used in the creation of single-bit locks; they work without the need for any additional memory barriers. 

  * There is a new `KERN_CONT` priority level for `printk()`. It is, in fact, empty; it is meant to serve as a marker for `printk()` calls which continue a previous (not terminated with a newline) printed line. 

  * The watchdog device drivers have been moved to a new home at `drivers/watchdog`. 

  * A notifier mechanism for console events has been added; this feature is aimed at accessibility tools (like Speakup) which need to know when something has changed on the console display. 

  * The filesystem export operations, used to make filesystems available over protocols like NFS, have been reworked. Two new methods (`fh_to_dentry()` and `fh_to_parent()`) replace the old `get_dentry()` interface. There is a new structure (`struct fid`) used to describe file handles. This work is aimed at making the export interface easier to use and (eventually) supporting 64-bit inode numbers. 

  * The [virtio](<http://lwn.net/Articles/239238/>) patches - providing an infrastructure for I/O into and out of virtualized guests \- have been merged. 

Now the stabilization period begins.
