---
title: Semantics of MMIO mapping attributes across architectures
url: https://lwn.net/Articles/698014/
date: "August 24, 2016"
category: "Device drivers-Support APIs"
author: "August 24, 2016 This article was contributed by Paul E. McKenney, Will Deacon, and Luis R. Rodriguez"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

August 24, 2016

This article was contributed by Paul E. McKenney, Will Deacon, and Luis R. Rodriguez

Although both [memory-mapped I/O](<https://en.wikipedia.org/wiki/Memory-mapped_I/O>) (MMIO) and normal memory (RAM) are ultimately accessed using the same CPU instructions, they are used for very different purposes. Normal memory is used to store and retrieve data, of course, while MMIO is instead primarily used to communicate with I/O devices, to initiate I/O transfers and to acknowledge interrupts, for example. And while concurrent access to shared memory can be [complex](<https://www.kernel.org/doc/Documentation/memory-barriers.txt>), programmers need not worry about what type of memory is in use, with only a few [exceptions](<https://en.wikipedia.org/wiki/Row_hammer>). In contrast, even in the single-threaded case, understanding the effects of MMIO read and write operations requires a detailed understanding of the specific device being accessed by those reads and writes. But the Linux kernel is not single-threaded, so we also need to understand MMIO ordering and concurrency issues. 

This article looks under the hood of the Linux kernel's MMIO implementation, covering a number of topics: 

  1. MMIO introduction
  2. MMIO access primitives
  3. Memory types
  4. x86 implementation
  5. ARM64 implementation
  6. PowerPC implementation
  7. Summary and conclusions 

#### MMIO introduction

![\[MMIO write\]](https://static.lwn.net/images/2016/MMIOwrite.svg)

MMIO offers both read and write operations. MMIO writes are used for one-way communication, causing the device to change its state, as shown in the diagram on the right. The MMIO write operation transmits the data and a portion of the address to the device, and the device uses both quantities to determine how it should change its state. 

**Quick quiz 1** : Why can't the device make use of the full address?   
Answer

The size of the MMIO write is also significant, in fact, the device might react completely differently to a single-byte MMIO write than to (say) a four-byte MMIO write. The size of the access could therefore be thought of as additional bits feeding into the device's state-change logic. 

MMIO reads are used for two-way communication, causing the device to return a value based on its current state. ![\[MMIO read\]](https://static.lwn.net/images/2016/MMIOread.svg) The MMIO read operation transmits a portion of the address to the device, which the device can use to determine how to query its state in order to compute the return value. Interestingly enough, the device can also change its state based on the read, and many devices do exactly that. For example, the MMIO read operation that reads a character from a serial input device would be expected to also remove that character from the device's internal queue, so that the next MMIO read would read the next input character. As with writes, the size of the MMIO read is significant. 

An MMIO read operation signals its completion by returning the value from the device. In contrast, the only way to determine when an MMIO write operation has completed is to do an MMIO read to poll for completion. Such polling is completely and utterly device-dependent, sometimes even requiring time delays between the initial MMIO write and the first subsequent MMIO read. 

Given that both MMIO reads and writes can change device state, ordering is extremely important and, as will be discussed below, many of the Linux kernel's MMIO access functions provide strong ordering, both with each other and with locking primitives and value-returning atomic operations. However, for devices such as frame buffers, it is not helpful to provide strict ordering, as the order of writes to independent pixels is irrelevant. In fact, [write combining](<https://en.wikipedia.org/wiki/Write_combining>) (WC) is an important frame-buffer performance optimization, and this optimization explicitly ignores the order of non-overlapping writes. This means that the hardware and the Linux kernel need some way of specifying which MMIO locations can and cannot tolerate reordering. 

The x86 family responded to this need with [ memory type range registers](<https://en.wikipedia.org/wiki/Memory_type_range_register>) (MTRRs), which were used to set WC cache attributes for [VGA](<https://en.wikipedia.org/wiki/Video_Graphics_Array>) memory. MTRRs have been used to enable different caching policies for memory regions on different [PCI](<https://en.wikipedia.org/wiki/Conventional_PCI>) devices. When an x86 system boots, the default cache attributes for physical memory are set by setting the model-specific register that sets the default memory type (MSR_MTRRdefType), and then MTRRs are used to modify memory in other ranges to other cache attributes, for example, uncached (UC) or WC for MMIO regions. Some BIOSes set MSR_MTRRdefType to [writeback](<https://en.wikipedia.org/wiki/Cache_\(computing\)#WRITE-BACK>), which is a common default for DRAM. Other BIOSes might set MSR_MTRRdefType to UC, and then use MTRRs to set DRAM to writeback. One of the biggest issues with MTRRs is the limited number of them. In addition, using MTRRs on x86 requires the use of the heavyweight `stop_machine()` call whenever the MTRR configuration changes. 

**Quick quiz 2** : Can you set up UC access to normal (non-MMIO) RAM?   
Answer

The [ page attribute table](<https://en.wikipedia.org/wiki/Page_attribute_table>) (PAT) relies on paging to lift this limitation. Unfortunately, the BIOS runs in [real mode](<https://en.wikipedia.org/wiki/Real_mode>), in which paging is not available, which means that the BIOS must continue to use MTRRs so x86 systems will continue to have them. However, Linux kernels _can_ use paging, and can therefore use PAT when running on hardware providing it. 

In short, MMIO reads and writes can be thought of as a message-passing communication mechanism for interacting with devices; they can be uncached for traditional device access or write combining for access to things like frame buffers and [InfiniBand](<https://en.wikipedia.org/wiki/InfiniBand>), and they require special attention in order to interact properly with synchronization primitives such as locks. The Linux kernel therefore provides architecture-specific primitives that implement MMIO accesses, as described in the next section. 

#### MMIO access primitives

The `readX()` function does MMIO reads, with the `X` specifying the size of the read, so that `readb()` reads one byte, `readw()` reads two bytes, `readl()` reads four bytes, and, on some 64-bit systems, `readq()` reads eight bytes. These functions are all little-endian, but in some cases, big-endian behavior can be specified using an additional "`_be`" component to the `X` suffix. 

The `writeX()` function does MMIO writes, with the `X` specifying write size as above, and again in some cases with an additional "`_be`" component to the `X` suffix. 

There are also `inX()` and `outX()` functions that map back to the x86 `in` and `out` instructions, respectively. The `X` suffix contains the size in bits and a "`be`" or "`le`" component to specify endianness. These are sometimes mapped to MMIO on non-x86 systems. The `ioreadX()` and `iowriteX()`, where `X` is the number of bits to operate on, can also be used to read and write MMIO; they were added to [hide the differences](<https://lwn.net/Articles/102232/>) between `in`/`out` operations and MMIO. 

Linus Torvalds created the following example on a Kernel Summit whiteboard to illustrate ordering requirements: 

```
1 unsigned long global = 0;
         2
         3 void locked_device_output(void)
         4 {
         5   spin_lock(&a);
         6   i = global++;
         7   writel(i, dev_slave_address);
         8   spin_unlock(&a);
         9 }
```

Line 5 acquires lock `a`, line 6 increments a global variable `global` under that lock, line 7 writes the previous value of `global` to an MMIO location at `dev_slave_address`, and finally line 8 releases the lock. In an ideal world, both lines 6 and 7 would be protected by the lock when `locked_device_output()` is invoked concurrently. 

Of course, the normal variable `global` is protected by the lock, that being what locks are for. However, for the MMIO write to `dev_slave_address`, such protection requires that the implementations of `spin_lock()`, `spin_unlock()`, and `writel()` [cooperate](<http://yarchive.net/comp/linux/write_barriers.html>) so as to provide the ordering required to force the `writel()` to `dev_slave_address` of the value 0 from the first `locked_device_output()` call to happen before the second call writes the value 1\. This is what x86 does and what most developers would expect to happen. Weakly ordered systems must therefore insert whatever memory barriers are required to enforce this ordering. 

**Quick quiz 3** : What do weakly ordered systems do to enforce this ordering?   
Answer

Providing the required ordering can be expensive on weakly ordered systems. Because there are a number of situations where ordering is not required (for example, frame buffers), the Linux kernel provides relaxed variants (`readX_relaxed()` corresponding to `readX()` and `writeX_relaxed()` corresponding to `writeX()`) that do not guarantee strong ordering, which can be used as follows: 

```
1 unsigned long global = 0;
         2
         3 void locked_device_output(void)
         4 {
         5   spin_lock(&a);
         6   i = global++;
         7   writel_relaxed(i, dev_slave_address);
         8   spin_unlock(&a);
         9 }
```

Because this example uses `writel_relaxed()` instead of `writel()`, the `writel_relaxed()` can be reordered with the `spin_unlock()`, so that the write of the value 1 might well precede the write of the value 0. An MMIO write memory barrier, called `mmiowb()`, may be used to prevent MMIO writes from being reordered with each other or or with locking primitives and value-returning atomic operations. This `mmiowb()` primitive can be used as shown below: 

```
1 unsigned long global = 0;
         2
         3 void locked_device_output(void)
         4 {
         5   spin_lock(&a);
         6   i = global++;
         7   writel_relaxed(i, dev_slave_address);
         8   mmiowb();
         9   spin_unlock(&a);
        10 }
```

Again, without the `mmiowb()`, the `writel_relaxed()` call might be reordered with its counterpart from a later instance of this critical section that was running on a different CPU. 

**Quick quiz 4** : Why can't `mmiowb()` be used with `writel()`?   
Answer

A more useful version of this example might do several `writel_relaxed()` invocations in the critical section followed by a final `mmiowb()`. It is worth noting that `mmiowb()` is a no-op on most architectures. 

However, `_relaxed()` accesses from a given CPU to a specific device are guaranteed to be ordered with respect to each other. Tighter semantics can of course be used: per-bus or even global, for example. 

Nevertheless, the `_relaxed()` functions are not primitives that most device driver developers normally consider using and, even if they did, there are still some kernel calls, such as locking calls, that might nullify such relaxed effects. Its unclear if these implications have always been well thought-out throughout the entire kernel. For instance, it is now understood that PowerPC's default kernel `writel()` uses a memory barrier; although typically one would expect write-combining to happen in user space for frame buffers, kernel writes could [nullify](<http://marc.info/?l=linux-fbdev&m=143519363119441&w=2>) the write-combining effects. 

Asking more developers to use the relaxed primitives when write combining might be the first instinct to address this situation, there are other possible issues which still need to be considered, however. For instance, would using a `spin_lock()` nullify any write-combining effects on some architectures even if relaxed primitives were used? If so, which architectures would be affected? Are we nullifying write combining in some areas in the kernel even on x86 if locks are used? To answer these questions, we must review each architecture's MMIO and locking primitive helpers and the implications of them on ordering. 

#### Memory types

As noted earlier, this article covers the two most common flavors of MMIO, uncached MMIO and write-combining MMIO. In uncached MMIO, each read and write is independent and in some sense atomic, with no combining, prefetching, or caching of any kind. The `ioremap_nocache()` function is used to map uncached MMIO registers. In write-combining MMIO, both reads and writes can be both coalesced and reordered, even the non-`_relaxed()` reads and writes. Memory that is write combining is also normally "prefetchable", and these terms sometimes appear to be used interchangeably. The `ioremap_wc()` function is used to map write-combining MMIO registers. 

For any given architecture, there are some questions about write combining: 

  * What prevents reordering and combining? (Presumably `mmiowb()` and perhaps also `mb()`.) 
  * What operations flush the write buffers? (Hardware dependent, but reads from a given device typically flush prior writes to that same device.) 
Five additional per-architecture questions will be addressed in tabular form: 
  1. Must non-relaxed accesses to MMIO regions be confined to lock-based critical sections? (Presumably the answer is "yes".) 
  2. Must relaxed accesses to MMIO regions be confined to lock-based critical sections? (Prudence would suggest "yes" as the answer, at least in the UC case.) 
  3. Must reads from MMIO regions be ordered with each other? (Presumably the answer is "no" for `_relaxed()` primitives.) 
  4. Must reads from MMIO regions be ordered with writes to other locations within the region? (Presumably the answer is "no".) 
  5. Must accesses to specific locations in MMIO regions be ordered with other accesses to that same location? (Presumably the answer is "yes", even for accesses to WC MMIO regions, at least for completely overlapping updates. Otherwise you would get old pixels on your display, after all.) 

It is natural to wonder what happens if a given range of MMIO registers is mapped as write combining at one virtual address and as uncached (non-write-combining) at some other address. The answer varies across both architectures and devices, so that the current Linux-kernel stance is "don't do that" unless absolutely necessary. 

Regardless of what the answers are, they clearly need to be better documented. 

#### Existing practice

There are more than 2,000 uses of `writel_relaxed()` and more than 1,000 uses of `readl_relaxed()`, so existing practice must be taken into account: changes might be made, but not lightly. Many uses of these primitives are in architecture-specific code, but there are common-code uses in some drivers. We took a look at a few of them: 

  * `drivers/ata/ahci_brcmstb.c`: This driver uses `brcm_sata_readreg()` and `brcm_sata_writereg()` to wrap `readl_relaxed()` and `writel_relaxed()`, respectively. The code appears to expect that relaxed reads and writes from/to the same device will be ordered. 
  * `drivers/crypto/atmel-aes.c`: This driver uses `atmel_aes_read()` and `atmel_aes_write()` to wrap `readl_relaxed()` and `writel_relaxed()`, respectively. The code appears to expect that relaxed reads and writes from/to the same device will be ordered. 
  * `drivers/crypto/img-hash.c`: This driver uses `img_hash_read()` and `img_hash_write()` to wrap `readl_relaxed()` and `writel_relaxed()`, respectively. The code appears to expect that relaxed reads and writes from/to the same device will be ordered. 
  * `drivers/crypto/ux500/cryp/cryp.c` appears to expect that relaxed reads and writes from/to the same device will be ordered. At present, this driver does not seem to be used outside of ARM, but crypto IP blocks are not necessarily tied to ARM. 

Although the bulk of the uses of the relaxed I/O accessors are confined to one architecture or another, it would not necessarily be wise to define CPU-family-specific changes to their semantics. Such changes are likely to cause serious problems should one of the corresponding hardware IP blocks ever be used by an implementation of the some other CPU family. 

#### x86 implementation

The `x86` mapping is as follows:  API | Implementation | Ordering  
---|---|---  
`mmiowb()` | `barrier()` | Provided by x86 ordering  
`spin_unlock`() | `arch_spin_unlock()` | Provided by x86 ordering  
`inb()  
inw()  
inl()` | `inb` instruction  
`inw` instruction  
`inl` instruction | See table below  
`outb()  
outw()  
outl()` | `outb` instruction  
`outw` instruction  
`outl` instruction | See table below  
`readb()`, `readb_relaxed()`, `ioread8()  
readw()`, `readw_relaxed()`, `ioread16()  
readl()`, `readl_relaxed()`, `ioread32()  
readq()`, `readq_relaxed()` | MMIO read | See table below  
`writeb()`, `writeb_relaxed()`, `iowrite8()  
writew()`, `writew_relaxed()`, `iowrite16()  
writel()`, `writel_relaxed()`, `iowrite32()  
writeq()`, `writeq_relaxed()` | MMIO write | See table below  
  
The `readX()` and `writeX()` definitions are built by the `build_mmio_read()` and `build_mmio_write()` macros, respectively. 

The x86 answers to the other questions appear to be as follows, based on a scan through "[Intel 64 and IA-32 Architectures Software Developer's Manual V3](<http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html>)": 

  * What prevents reordering and combining? For non-write-combining MMIO regions, everything. For write-combining MMIO regions, `mmiowb()`, `smp_mb()`, an access to a non-write-combining MMIO region, an interrupt, or a locked instruction, which is an instruction having the `LOCK` prefix that signals that the instruction is to be an atomic read-modify-write instruction. 
  * What operations flush the write buffers? The same operations that prevent reordering and combining. 
x86 | Within `ioremap_wc()` | Within `ioremap_nocache()` | Against normal memory  
---|---|---|---  
`_relaxed()` | Unordered, ordered to same location. | Ordered. | Ordered.  
non-`relaxed()` | Unordered, ordered to same location. | Ordered. | See [*] below.  
  
[*]: | Accesses to `ioremap_wc()` memory are not ordered with accesses to normal memory unless: 

  1. Either there is an intervening `smp_mb()`, or 
  2. The normal-memory access uses the `lock` prefix, **and**
  3. The I/O fabric is "sane" in that it avoids reordering and buffering invisible to the CPU. I/O fabrics that have multiple layers of I/O bus are all too often _not_ sane. 
  
---|---  
  
**Quick quiz 5** : But if x86 always uses the same instructions for MMIO, how can the ordering semantic differ for `ioremap_wc()` and `ioremap_nocache()` regions?   
Answer

To reiterate that last point, note that this all assumes sane hardware. It is possible to construct x86 systems with I/O bus structures that do not follow the above rules. Drivers written for such systems typically need to "confirm" prior MMIO writes by doing a later MMIO read that either forces ordering or verifies the state changes caused by the write. The exact confirmation method will depend on the details of the I/O device in question. 

On older MTRR-only x86 systems, some frame-buffer drivers must also use `arch_phys_wc_add()`, because on such systems `ioremap_wc()` would otherwise produce an uncached non-write-combining mapping for the corresponding device. This inability of `ioremap_wc()` to Do The Right Thing can be due to limited numbers of MTRRs, limited MTRR size, I/O-mapping alignment constraints, page aliasing (for example, to provide both kernel- and user-mode access to MMIO registers), and because some old hardware simply cannot be shoehorned into the nice new PAT-based Linux-kernel APIs. 

For more details, see commits ["drivers/video/fbdev/atyfb: Use arch_phys_wc_add() and ioremap_wc()"](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=7d89a3cb159a>), ["drivers/video/fbdev/atyfb: Clarify ioremap() base and length used"](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=f55de6ec375d>), and ["drivers/video/fbdev/atyfb: Carve out framebuffer length fudging into a helper"](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=eacd2d542610>). Commit ["drivers/video/fbdev/atyfb: Replace MTRR UC hole with strong UC"](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=3cc2dac5be3f>) is particularly instructive, as it describes how old hacks were replaced by newer less-hacky hacks based on the new API. 

Those interested in page aliasing should refer to [Documentation/ia64/aliasing.txt](<https://www.kernel.org/doc/Documentation/ia64/aliasing.txt>), particularly the "POTENTIAL ATTRIBUTE ALIASING CASES" section. Fortunately, most device manufacturers now dedicate one full PCI base address register ([BAR](<http://wiki.osdev.org/PCI#Base_Address_Registers>)) to MMIO and another for frame-buffer use, which means that developers writing drivers for modern devices can for the most part simply use the `ioremap_nocache()` and `ioremap_wc()` APIs. 

One important last note: On x86 systems, spinlock-release primitives usually use a plain store instruction. This will not order accesses within `ioremap_wc()` regions. Although this might seem strange at first glance, it has the advantage that the effectiveness of write combining is not limited by `spin_unlock()` invocations. 

#### ARM64 implementation

The `arm64` mapping is as follows:  API | Implementation | Ordering  
---|---|---  
`mmiowb()` | do { } while (0) | Provided by ARM64 ordering  
`spin_unlock`() | `arch_spin_unlock()` | `llsc` or `lse` instruction  
`readb()`, `ioread8()  
readw()`, `ioread16()  
readl()`, `ioread32()  
readq()` | MMIO read | Follow MMIO read by `rmb()`  
`writeb()`, `iowrite8()  
writew()`, `iowrite16()  
writel()`, `iowrite32()  
writeq()` | MMIO write | Precede MMIO write with `wmb()`  
`readb_relaxed()  
readw_relaxed()  
readl_relaxed()  
readq_relaxed()` | MMIO read | See table below  
`writeb_relaxed()  
writew_relaxed()  
writel_relaxed()  
writeq_relaxed()` | MMIO write | See table below  
  
Note that although ARM does distinguish between WC and non-WC flavors of MMIO regions in terms of ordering; the type of accessor (`_relaxed()` vs. non-`_relaxed()`) also has a big role to play. Note that ARM64's non-`_relaxed()` accessors have ordering properties similar to [total store order](<https://en.wikipedia.org/wiki/Processor_consistency#Similarity_to_SPARC_V8_TSO.2C_IBM-370.2C_and_x86-TSO_Memory_Models>) (TSO), that is, they order prior reads against later reads and writes, and also order prior writes against later writes, but they do not order prior writes against later reads. 

The ARM64 answers to the questions are as follows: 

  * What prevents reordering and combining? A non-relaxed MMIO access (aside from not ordering prior writes against later reads) or either `mb()`, `rmb()`, or `wmb()`. 
  * What operations flush the write buffers for write-combining regions? Either `mb()` or `wmb()`. But please note that this flushing has effect only within the CPU. These memory barrier do not necessarily affect any write buffers that might reside on external I/O buses. 
ARM64 | Within `ioremap_wc()` | Within `ioremap_nocache()` | Against normal memory  
---|---|---|---  
`_relaxed()` | Unordered, but fully ordered for accesses to the same address. | Unordered, but fully ordered for accesses to same device. | Unordered.  
non-`relaxed()` | "TSO", but fully ordered for accesses to the same address. | "TSO", but fully ordered for accesses to same device. | See below.  
  
In the above table, "TSO" allows prior writes to be reordered with later reads, but prevents any other reordering. 

The lower right-hand cell's rules are as follows:  Prior Access | Next Access | Ordering  
---|---|---  
Non-Relaxed Read | Plain Read | Ordered (useful for reading from a DMA buffer).  
Non-Relaxed Read | Plain Write | Ordered.  
Non-Relaxed Write | Plain Read | Unordered.  
Non-Relaxed Write | Plain Write | Unordered.  
Plain Read | Non-Relaxed Read | Unordered (departure from TSO).  
Plain Read | Non-Relaxed Write | Unordered (departure from TSO).  
Plain Write | Non-Relaxed Read | Unordered.  
Plain Write | Non-Relaxed Write | Ordered (useful for triggering DMA).  
  
Just as with x86, it is possible to construct ARM systems with I/O bus structures that do not follow the above rules. Drivers written for such systems typically need to "confirm" prior MMIO writes by doing a later MMIO read that either forces ordering or verifies the writes' state changes. The exact confirmation method will depend on the details of the I/O device in question. 

#### PowerPC implementation

Finally, the PowerPC mapping uses an `->io_sync` field in the Linux kernel's PowerPC-specific per-CPU data. This field is set by PowerPC MMIO writes, and tested at unlock time. If this field is set, the unlock primitive executes a heavyweight `sync` instruction, which forces the last MMIO write to be contained within the critical section. 

The mapping is as follows:  API | Implementation | Ordering  
---|---|---  
`mmiowb()` |  | `sync` and clear `->io_sync`  
`spin_unlock`() | `arch_spin_unlock()` | If `->io_sync` set, `sync` and clear `->io_sync`  
`in_8()  
in_be16()  
in_be32()  
in_le16()  
in_le32()` | MMIO read | `sync` followed by read followed by `twi;isync`  
`out_8()  
out_be16()  
out_be32()  
out_le16()  
out_le32()` | MMIO write | `sync` followed by write followed by set `->io_sync`  
`readb()`, `inb()`, `ioread8()  
readw()`, `inw()`, `ioread16()  
readw_be()`, `ioread16be()  
readl()`, `inl()`, `ioread32()  
readl_be()`, `ioread32be()  
readq()  
readq_be()` | `in_8()  
in_le16()  
in_be16()  
in_le32()  
in_be32()  
in_le64()  
in_be64()` | `sync` followed by read followed by `twi;isync`  
`writeb()`, `outb()`, `iowrite8()  
writew()`, `outw()`, `iowrite16()  
writew_be()`, `iowrite16be()  
writel()`, `outl()`, `iowrite32()  
writel_be()`, `writel_be()  
writeq()  
writeq_be()` | `out_8()  
out_le16()  
out_be16()  
out_le32()  
out_be32()  
out_le64()  
out_be64()` | `sync` followed by write followed by set `->io_sync`  
  
The alert reader will note the duplication of some names in the "API" and "Implementation" columns, for example, `in_8()`. The definitions are in `arch/powerpc/include/asm/io.h` and `arch/powerpc/include/asm/io-defs.h`. 

Other implementation strategies are possible, of course. One approach would be for `mmiowb()` and `arch_spin_unlock()` to both unconditionally execute the `sync` instruction and to dispense with the `->io_sync` flag. Another approach would be to make `mmiowb()` an no-op, eliminate the test and `sync` instruction from `arch_spin_unlock()`, and replace setting of `->io_sync` by a `sync` instruction. However, both of these approaches would greatly increase the number of executions of the expensive `sync` instruction, so, for PowerPC, the implementation in the above table is preferred. 

Currently (v4.3) PowerPC's `_relaxed()` interfaces operate exactly the same as do their non-relaxed counterparts. Part of the motivation for the MMIO discussion during the technical day at the [2015 Linux Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>) was to determine how and to what extent PowerPC could actually relax the `_relaxed()` implementations. However, this article limits itself to documenting current reality. 

The PowerPC answers to the questions appears to be as follows: 

  * What prevents reordering and combining? Any MMIO access, `mmiowb()`, or `smp_mb()`. This is not a good thing, as it makes for slow frame buffers. 
  * What operations flush the write buffers? The same operations that prevent reordering and combining. 
PowerPC | Within `ioremap_wc()` | Within `ioremap_nocache()` | Against normal memory  
---|---|---|---  
`_relaxed()` | Fully Ordered. | Fully Ordered. | Fully Ordered.  
non-`relaxed()` | Fully Ordered. | Fully Ordered. | Fully Ordered.  
  
#### Summary and conclusions

MMIO should be thought of as a message-passing mechanism that communicates with hardware rather than a variant of normal memory. As such, MMIO is not only device-specific, but also specific to the hardware path between the CPU and the device. In the general case, which includes ill-considered hardware designs, even memory barriers cannot always order accesses: in some cases, the device's state must be polled to determine when a prior access has completed. 

Nevertheless, the Linux kernel offers a rich set of primitives with which to interact with MMIO devices, and this article has given a brief overview of how they work and how they may be used. 

#### Acknowledgments

We are grateful to Michael Ellerman, Gautham Shenoy, Peter Zijlstra, Andy Lutomirski, and Boqun Feng for their review and comments. We owe thanks to Toshimitsu Kani, Dave Airlie, Christoph Hellwig, and Matt Fleming for a number of important discussions, and to Jim Wasko for his support of this effort. 

####  Answers to Quick quizzes

**Quick quiz 1** : Why can't the device make use of the full address? 

**Answer** : Because part of the address is used to select the device. 

**Back to Quick quiz 1**.

**Quick quiz 2** : Can you set up UC access to normal (non-MMIO) RAM? 

**Answer** : You can, and this is in fact actually used for GPU memory for GPUs that cannot snoop the CPU caches. One example may be found in `ati_create_page_map()`, which uses `__get_free_page()` to allocate a page of DRAM and then later uses `set_memory_uc()` to change the cache attribute. There is also a `set_memory_wc()`. Although `set_memory_uc()` and `set_memory_wc()` may also be used to set up MMIO, such use is likely to be strongly discouraged. In addition, it is quite possible that the `set_memory_uc()` and `set_memory_wc()` APIs will change. 

**Back to Quick quiz 2**.

**Quick quiz 3** : What do weakly ordered systems do to enforce this ordering? 

**Answer** : They enforce this ordering by a combination of hardware and software ordering constraints. Please read on for more information, leading up to descriptions of the ARM64 and the PowerPC implementations. 

**Back to Quick quiz 3**.

**Quick quiz 4** : Why can't `mmiowb()` be used with `writel()`? 

**Answer** : Actually, they really can be used together. But there is little point in doing so because `writel()` already provides strong ordering. Therefore, placing an `mmiowb()` after a `writel()` has no effect other than to slow things down. 

Of course, in this case it would be simpler to just use `writel()` instead of both `writel_relaxed()` and `mmiowb()`. However, `mmiowb()` is quite useful when there are multiple `writel_relaxed()`, all of which need to be contained within the critical section. A single `mmiowb()` placed between the last `writel_relaxed()` and the unlock will contain all of them, and with the added memory-barrier overhead incurred only once at `mmiowb()` time instead of once for each and every `writel()`. 

**Back to Quick quiz 4**.

**Quick quiz 5** : But if x86 always uses the same instructions for MMIO, how can the ordering semantic differ for `ioremap_wc()` and `ioremap_nocache()` regions? 

**Answer** : Because the ordering is controlled not by the instructions, but rather by the MTRR settings (in older systems) or by PAT (in newer systems). 

**Back to Quick quiz 5**.
