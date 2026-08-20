---
title: More stuff for 2.6.25
url: https://lwn.net/Articles/267849/
date: "February 6, 2008"
category: "Releases-2.6.25"
author: "By Jonathan Corbet February 6, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 6, 2008

Since [last week's installment](<http://lwn.net/Articles/266721/>), some 3800 changesets have been merged into the mainline git repository. Some of the more interesting user-visible changes found in that patch stream include: 

  * Support for new hardware, including RDC R-321x system-on-chip processors, Onkyo SE-90PCI and SE-200PCI sound devices, Xilinx ML403 AC97 controllers, TI TLV320AIC3X audio codecs, Realtek ALC889/ALC267/ALC269 codecs, VIA VT1708B HD audio codecs, SiS 7019 Audio Accelerator devices, C-Media 8788 (Oxygen) audio chipsets, Asus AV200-based sound cards, Freescale MPC8610 audio devices, Audiotrak Prodigy 7.1 HiFi audio devices, Conexant 5051 audio codecs, MediaTek/TempoTec HiFier Fantasia sound cards, wireless RNDIS devices (and Broadcom 4320-based devices in particular), USB printer gadgets (intended for use in printer firmware), and NetEffect 1/10Gb ethernet adapters. 

  * The nearly-unused ALSA sequencer instrument layer has been removed. 

  * SELinux has a new set of checks which allow the creation of policies which control the flow of packets into and out of the system. 

  * Netfilter has a more flexible "hashlimit" mechanism for limiting the number of packets to/from a given source over time. 

  * There is a new "flow" classifier for the network fair queueing code which allows the more flexible creation of traffic policies. 

  * The futex mechanism has a new "bitset wait" mechanism which allows for more targeted wakeups. This feature will be used by glibc to implement optimized reader-writer locks. 

  * PCI hotplug is no longer an experimental feature. 

  * Support for PCI Express ASPM, a power management protocol, has been added. 

  * The virtio "balloon" driver (which can be used to change the amount of memory used by a KVM guest) and PCI driver have been added. 

  * The `CLONE_STOPPED` bit (for the `clone()` system call) is said to be unused and is planned for removal. For 2.6.25, a warning will be printed. 

  * The `timerfd()` system call is back, with a [reworked, more capable API](<http://lwn.net/Articles/260172/>). 

  * The [page map patches](<http://lwn.net/Articles/230975/>), which enable much better accounting of memory use by processes, have been merged. 

  * The "PM QOS" infrastructure allows both kernel and user-space code to register quality-of-service requirements (in the form of CPU DMA latency, network latency, and network throughput). These requirements will be taken into account when the kernel considers putting the system into a lower-power state. 

  * Per-process capability bounding sets (which permanently remove potential capabilities from a process) are now supported. 64-bit capability mask support has also been merged. 

  * The [simplified mandatory access control kernel](<http://lwn.net/Articles/244531/>) (SMACK) security module has been merged. 

  * The smbfs filesystem has (finally) been deprecated in favor of CIFS. It is now scheduled for removal in 2.6.27. 

  * There is a new RPC transport module allowing (client) NFS mounts using RDMA. 

Changes visible to kernel developers include: 

  * A large number of SUNRPC symbols (`rpc_`* and `rpcauth_`*) have been changed to GPL-only exports. 

  * The x86 architecture merger continues, with quite a few files being coalesced. 

  * The "flatmem" and "discontigmem" memory models have been removed on the 64-bit x86 architecture; "sparsemem" is now used for all builds. 

  * The x86 spinlock implementation has been replaced with a "ticket spinlock" mechanism which provides fair FIFO behavior. 

  * The `fastcall` function attribute didn't do anything on the x86 architecture, so it has been removed. 

  * x86 has a new set of functions for easily manipulating page attributes. They are: 

```
set_memory_uc(unsigned long addr, int numpages); /* Uncached */
            set_memory_wb(unsigned long addr, int numpages); /* Cached */
            set_memory_x(unsigned long addr, int numpages);  /* Executable */
            set_memory_nx(unsigned long addr, int numpages); /* Non-executable */
            set_memory_ro(unsigned long addr, int numpages); /* Read-only */
            set_memory_rw(unsigned long addr, int numpages); /* Read-write */
```

There is also a set of `set_pages_`* functions which take a `struct page` pointer rather than a beginning address. 

  * Early-boot debugging of x86 systems via the FireWire port is now supported. 

  * Bidirectional command support has been added to the SCSI layer. 

  * There is a new process state called `TASK_KILLABLE`. It is a blocked state similar to `TASK_UNINTERRUPTIBLE`, with the difference that a wakeup will happen upon delivery of a fatal signal. The idea is to allow (almost) uninterruptible sleeps, but to still allow the process to be killed outright - thus ending the problem of unkillable processes stuck in the "D" state. There is a new set of functions for using this state: `wait_event_killable()`, `schedule_timeout_killable()`, `mutex_lock_killable()`, etc. 

  * `add_disk_randomness()` has been unexported as there are no more in-tree users. 

  * `pci_enable_device_bars()` has been replaced by two more-specific functions: `pci_enable_device_io()` and `pci_enable_device_mem()`. 

  * The high-resolution timer API has been augmented with: 

```
unsigned long hrtimer_forward_now(struct hrtimer *timer,
                                              ktime_t interval);
```

It will move the given timer's expiration forward past the current time as determined by the associated clock. 

  * The `device` structure now holds a pointer to a `device_dma_parameters` structure: 

```
struct device_dma_parameters {
        	unsigned int max_segment_size;
        	unsigned long segment_boundary_mask;
            };
```

These parameters are used by the DMA mapping layer (and the IOMMU mapping code in particular) to ensure that I/O operations are set up within the device's constraints. The PCI layer supports this feature with two new functions: 

```
int pci_set_dma_max_seg_size(struct pci_dev *dev, unsigned int size);
            int pci_set_dma_seg_boundary(struct pci_dev *dev, unsigned long mask);
```

Drivers for devices with unusually strict DMA limitations should probably use these functions to ensure that those restrictions are respected. 

One thing which has _not_ made it into 2.6.25 is the KGDB debugger for the x86 architecture. Amusingly, a linux.conf.au kernel mini-conf discussion of "sneaking" KGDB past Linus proceeded for some time before the participants noticed him standing in the back of the room listening to the whole thing. His current [position](<https://lwn.net/Articles/267958/>) is that he won't pull it as part of the x86 tree, and he's still not much interested in the idea in general. 

As of this writing, the merge window is still open and could stay that way for as much as a week. So more interesting code could still find its way in through this merge window; stay tuned.
