---
title: The rest of the 6.14 merge window
url: https://lwn.net/Articles/1006378/
date: "February 3, 2025"
category: "Releases-6.14"
author: "By Jonathan Corbet February 3, 2025"
---

By **Jonathan Corbet**  
February 3, 2025

By the time that Linus Torvalds [released 6.14-rc1](<https://lwn.net/Articles/1007546/>) and closed the merge window for this development cycle, some 9,307 non-merge changesets had been pulled into the mainline repository — the lowest level of merge-window activity seen in years. There were, nonetheless, a number of interesting changes in the 5,000 commits pulled since [the first-half merge-window summary](<https://lwn.net/Articles/1005651/>) was written. 

First, though, to get an idea of how atypical this cycle is, here are the merge-window commit counts for recent development cycles: 

> Release| Commits  
> ---|---  
> 6.13| 12,142  
> 6.12| 12,115  
> 6.11| 12,954  
> 6.10| 12,402  
> 6.9| 13,342  
> 6.8| 13,052  
  
In other words, the norm for a long time is to have at least 30% more commits show up during the merge window than we saw this time. To find a slower merge window than this one, it is necessary to go back to the 3.6 release in 2012 (with 9,140 commits during the merge window) — though, the 4.0 release in 2015, with 9,591 merge-window commits, was close. The slowdown this time, which was broadly felt across kernel subsystems, may well be a result of the timing of the development cycle relative to the holiday season this year and not indicative of a longer-term trend; time will tell. 

It is also worth keeping some perspective in mind; as Torvalds said in the announcement: ""Of course, 'tiny' for us still means that there's half a million lines changed, and more than 10k commits in there"". 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The most interesting changes pulled in the latter part of the 6.14 merge window include: 

#### Architecture-specific

  * The OpenRISC architecture has gained support for [restartable sequences](<https://lwn.net/Articles/697979/>). 

#### Core kernel

  * There are two new BPF kfuncs — `bpf_local_irq_save()` and `bpf_local_irq_restore()` — that will disable interrupts on the local CPU and restore them again. They are intended to help with the implementation of interrupt-safe data structures in BPF programs. [This commit](<https://git.kernel.org/linus/4fec4c22f046>) includes a self-test program that demonstrates their use. 
  * The `MADV_DONTNEED` and `MADV_FREE` operations in [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) will now free the associated page tables (when possible) along with the indicated address range. For some workloads, empty page tables can waste a lot of memory; see [this commit](<https://git.kernel.org/linus/6375e95f381e>) for a description of the problem that was solved. 
  * The [ntsync subsystem](<https://lwn.net/Articles/961884/>), which provides a set of Windows NT synchronization primitives, was merged in 6.10 but marked "broken". All of the broken pieces are now fixed, and this subsystem is now fully enabled. The interface implemented by ntsync is documented in [Documentation/userspace-api/ntsync.rst](<https://docs.kernel.org/next/userspace-api/ntsync.html>). 

#### Filesystems and block I/O

  * The XFS realtime device is a special mode intended to provide consistent and predictable latency for filesystem operations. In 6.14 the realtime device has gained support for reverse mapping (mapping a device block to the file the contains it) and reflink (sharing blocks between files) operations. 
  * The fsnotify subsystem has a new event type called `FS_PRE_ACCESS`. If a listener has requested this access, it will be notified before content in a file is read, and that access will be held up until the listener responds. The intended purpose, it seems, is to help with the implementation of hierarchical storage systems, enabling the listener to fill in data from slow storage if need be. This operation requires the `CAP_SYS_ADMIN` capability; as [the merge message](<https://git.kernel.org/linus/8883957b3c9d>) notes: ""it is an administrator responsibility to make sure the userspace event handler doesn't do stupid stuff that can DoS the system"". 
  * The [uncached buffered I/O patches](<https://lwn.net/Articles/998783/>) have been merged. This block I/O option is meant to provide most of the performance benefits of direct I/O using the normal POSIX I/O API and without some of the pitfalls of direct I/O. 
  * The NFS server now supports [NFS4.2+ attribute delegation](<https://www.ietf.org/archive/id/draft-ietf-nfsv4-delstid-08.html>), matching the client support merged for 6.11. From [the merge message](<https://git.kernel.org/linus/f34b580514c9>): ""An attribute delegation permits an NFS client to manage a file's mtime, rather than flushing dirty data to the NFS server so that the file's mtime reflects the last write, which is considerably slower."" 
  * The NFS ["localio" functionality](<https://docs.kernel.org/next/filesystems/nfs/localio.html>) has been enhanced so that clients can perform direct I/O in cases where the client and server are on the same host. 
  * Work has [been underway for some time](<https://lwn.net/Articles/932079/>) to allow the filesystems in user space (FUSE) subsystem to use io_uring. That capability has landed for 6.14. There is some beginning documentation in [Documentation/filesystems/fuse-io-uring.rst](<https://docs.kernel.org/next/filesystems/fuse-io-uring.html>). 

#### Hardware support

  * **GPIO and pin control** : Qualcomm 8917 pin controllers and Mediatek MT7988 pin controllers. 
  * **Industrial I/O** : Texas Instruments OPT4060 RGBW color sensors and ROHM Semiconductor BD79703 digital-to-analog converters. 
  * **Input** : Intel touch host controllers. 
  * **Media** : ST Microelectronics STM32 camera serial interface controllers. 
  * **Miscellaneous** : Microchip inter-processor communication interfaces, Samsung Exynos mailboxes, Nuvoton MA35 SoC NAND controllers, ST Microelectronics STC3117 fuel gauges, Thunderbolt 3 interfaces in alternate mode, Qualcomm SM8750 interconnects, and AMD AE4DMA controllers. 
  * **Sound** : TI TAS2781 SPI HD-audio side codecs and Awinic Technology aw88083 amplifiers. 

#### Miscellaneous

  * The `perf` tool has, as usual, received a number of enhancements; see [this merge message](<https://git.kernel.org/linus/7685b334d1e4>) for the list. 

#### Security-related

  * Once upon a time, there was a patch series implementing the [`O_MAYEXEC` option to `open()`](<https://lwn.net/Articles/820000/>), which would cause the open to succeed only if system policy (permission bits, security modules, etc.) would allow the target file to be executed. Its purpose was to enable language interpreters (for Python, for example) to decide whether executing a given file of text is consistent with local security policies. This idea was later [reimplemented as the `trusted_for()` system call](<https://lwn.net/Articles/832959/>), but that one was [rejected](<https://lwn.net/Articles/890957/>) during the 5.18 merge window. 

In 2024, the idea [returned](<https://lwn.net/Articles/982085/>) in a form that has now (after yet more changes) been merged for 6.14. There is a new option (`AT_EXECVE_CHECK`) to [`execveat()`](<https://man7.org/linux/man-pages/man2/execveat.2.html>) that causes the call to check whether the indicated file would be allowed to execute (without actually executing it). There are two new securebits (accessible via [`prctl()`](<https://man7.org/linux/man-pages/man2/PR_GET_SECUREBITS.2const.html>)) that tell interpreters how they should handle input. The bit now named `EXEC_RESTRICT_FILE` instructs interpreters to use `AT_EXECVE_CHECK` on a file to decide whether to execute its contents, while `EXEC_DENY_INTERACTIVE` tells the interpreter to refuse to read interactive commands. Interpreters must be updated separately to implement that functionality. 

See [Documentation/userspace-api/check_exec.rst](<https://docs.kernel.org/next/userspace-api/check_exec.html>) for some more information. 

#### Internal kernel changes

  * There has been a significant reorganization of the code for the CRC32 and CRC-T10DIF algorithms; they can now be invoked directly from the library interfaces and no longer need involve the kernel's crypto layer. The result is simpler and more efficient code. 
  * Kernel modules will be signed using the SHA512 algorithm rather than SHA1 by default. 
  * The kernel is now able to [allocate and free "frozen" pages](<https://lwn.net/Articles/1000654/>) that have no meaningful reference count. This change will make the slab allocator a bit more efficient, but it also helps in the longer-term goal of shrinking the `page` structure. 
  * The new [`zpdesc` memory descriptor](<https://lwn.net/Articles/1002362/>) also serves that latter goal by acting as a replacement for `struct page` for zswap pages. 
  * More Rust bindings have been merged for dealing with PCI buses, platform drivers, [Open Firmware](<https://en.wikipedia.org/wiki/Open_Firmware>), and more. [According to Greg Kroah-Hartman](<https://git.kernel.org/linus/2ab002c755bf>): ""We are almost at the 'write a real driver in rust' stage now, depending on what you want to do"". Included in the merge are [a sample PCI driver](<https://git.kernel.org/linus/685376d18e9a>) and [a platform driver](<https://git.kernel.org/linus/b2e8a83242c0>) in Rust. 
  * There is [a new implementation](<https://lwn.net/Articles/986892/>) of the build-time code that generates symbol versions for loadable modules. It uses information from the DWARF debugging records rather than parsing the source directly; the main driver for this work is to enable symbol versioning for modules written in Rust. For the time being, at least, both the old and new implementations remain in the kernel, with a configuration option to choose between them. 

The 6.14 release is most likely to happen on March 23, though the possibility of a one-week delay is always there. As this release stabilizes, eyes will be on linux-next to see whether 6.15 reverts to the mean with regard to commit activity. 

(LWN subscribers can follow the [upcoming 6.14 release](</ksdb/releases/v6.14/>), perhaps comparing it with [3.6](</ksdb/releases/v3.6/>) and [4.0](</ksdb/releases/v4.0/>), in the LWN Kernel Source Database).
