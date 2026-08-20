---
title: Shrinking the kernel with a hammer
url: https://lwn.net/Articles/748198/
date: "March 2, 2018"
category: Embedded systems
author: "March 2, 2018 This article was contributed by Nicolas Pitre"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

March 2, 2018

This article was contributed by Nicolas Pitre

This is the fourth article of [a series discussing various methods of reducing the size](<https://lwn.net/Articles/741494/>) of the Linux kernel to make it suitable for small environments. Reducing the kernel binary has its limits and we have pushed them as far as possible at this point. Still, our goal, which is to be able to run Linux entirely from the on-chip resources of a microcontroller, has not been reached yet. This article will conclude this series by looking at the problem from the perspective of making the kernel and user space fit into a resource-limited system.

A microcontroller is a self-contained system with peripherals, memory, and a CPU. It is typically small, inexpensive, and has low power-consumption characteristics. Microcontrollers are designed to accomplish one task and run one specific program. Therefore, the dynamic memory content of a microcontroller is usually much smaller than its static content. This is why it is common to find microcontrollers equipped with many times more ROM than RAM.

For example, the [ATmega328](<https://www.microchip.com/wwwproducts/en/ATmega328>) (a popular [Arduino](<https://www.arduino.cc/>) target) comes with 32KB of flash memory and only 2KB of static memory (SRAM). Now for something that can boot Linux, the [STM32F767BI](<http://www.st.com/en/microcontrollers/stm32f767bi.html>) comes with 2MB of flash and 512KB of SRAM. So we'll aim for that resource profile and figure out how to move as much content as possible from RAM to ROM.

#### Kernel XIP

The idea of eXecute-In-Place (XIP) is to have the CPU fetch instructions directly from the ROM or flash memory where it is stored and avoid loading them into RAM altogether. XIP is a given in the microcontroller world where RAM is small, as we've seen. But XIP is used a bit less on larger systems where RAM is plentiful and simply executing everything from RAM is often simpler; executing from RAM is also faster due to high-performance caches. This is why most Linux targets don't support XIP. In fact, XIP in the kernel appears to be supported only on ARM, and its introduction predates the Git era.

For kernel XIP, it is necessary to have ROM or flash memory directly accessible in a range of the processor's memory address space, alongside system RAM, without the need for any software drivers. [NOR flash](<https://en.wikipedia.org/wiki/Flash_memory#NOR_memories>) is often used for that purpose as it offers random access, unlike the block-addressed [NAND flash](<https://en.wikipedia.org/wiki/Flash_memory#NAND_memories>). Then, the kernel must be specially linked so the text and read-only data sections are allocated in the flash address range. All we need to do is enable `CONFIG_XIP_KERNEL` and the build system will prompt for the desired kernel physical address location in flash. Only the writable kernel data will be copied to RAM.

It is therefore highly desirable with an XIP kernel to have as much code and data as possible placed in flash memory. The more that remains in flash, the less will be copied to the precious RAM. By default, functions are put in flash, along with any data annotated with the `const` qualifier. It is convenient that all the "constification" work that took place in recent kernel releases, mainly for hardening purposes, directly benefits the XIP kernel case too.

#### User-space XIP and filesystems

User space is a huge consumer of RAM. But, just like the kernel, user-space binaries have read-write and read-only segments. It would be nice to have the user-space read-only segments stored in the same flash memory, and executed directly from there rather than being loaded into RAM. However, unlike the kernel, which is a static binary loaded or mapped only once from well-known ROM and RAM addresses, user-space executables are organized into a filesystem, making things more complicated.

Could we get rid of the filesystem? Certainly we could. In fact, this is what most small realtime operating systems do: they link their application code directly with the kernel, bypassing the filesystem layer entirely. And that wouldn't be completely revolutionary even for Linux, as kernel threads are more or less treated like user-space applications: they have an execution context of their own, they are scheduled alongside user applications, they can be signaled, they appear in the task list, etc. And kernel threads have no filesystem under them. An application made into a kernel thread could crash the entire kernel, but in a microcontroller environment lacking a memory-management unit (MMU), this is already the case for pure user-space applications.

However, having a filesystem around for user-space applications still has many advantages we don't want to lose:

  * Compatibility with full-fledged Linux systems, so our application can be developed and tested natively on a workstation; 

  * The convenience of having multiple unrelated applications together; 

  * The ability to develop and update the kernel and user space independently of each other; 

  * A clear boundary that identifies application code as [not being a derived work](<https://www.kernel.org/pub/linux/kernel/COPYING>) of the kernel in the context of the GPL. 

This being said, we want the smallest and simplest filesystem possible. Let's not forget that our flash memory budget is only 2MB, and our kernel (see [the previous article](<https://lwn.net/Articles/746780/>) in this series) weighs about 1MB already. That pretty much rules out writable filesystems due to their inherent overhead, and we don't want to be writing to the same flash where the kernel and user space live as this would render all the flash content inaccessible during write operations and crash any code executing from it.

Side note: It is possible to write to the actual flash memory being used for XIP with `CONFIG_MTD_XIP` but this is tricky, currently available only for [Intel](<https://elixir.bootlin.com/linux/v4.15/source/drivers/mtd/chips/cfi_cmdset_0001.c#L1065>) and [AMD](<https://elixir.bootlin.com/linux/v4.15/source/drivers/mtd/chips/cfi_cmdset_0002.c#L912>) flash memory, and requires [target-specific support](<https://elixir.bootlin.com/linux/v4.15/source/include/linux/mtd/xip.h>).

So our choices for small, read-only filesystems are:

  * [Squashfs](<https://www.kernel.org/doc/Documentation/filesystems/squashfs.txt>): highly scalable, compressed by default, somewhat complex code, no XIP support 

  * [Romfs](<https://www.kernel.org/doc/Documentation/filesystems/romfs.txt>): small and simple code, no compression, partial (only on systems without an MMU) XIP support 

  * [Cramfs](<https://www.kernel.org/doc/Documentation/filesystems/cramfs.txt>): small and simple code, compressed, partial (MMU-only) out-of-tree XIP support 

I settled on cramfs as the small amount of available flash memory warrants compression that romfs doesn't have, and cramfs's simple code base made it easier to add no-MMU XIP support quickly more than it would be for squashfs. Also, cramfs can be used with the block-device subsystem configured out entirely.

However the [early attempts](<https://lwn.net/Articles/235532/>) at adding XIP to cramfs were rather crude and lacking in a fundamental way. It was an all-or-nothing affair: each file was either completely uncompressed for XIP purposes, or entirely compressed. In reality, executables are made of both code and data, and since writable data has to be copied to RAM anyway, it is wasteful to keep that part uncompressed in flash. So I took upon myself to completely redesign cramfs XIP support for both the MMU and no-MMU cases. I included the needed ability to mix compressed and uncompressed blocks of arbitrary alignments, and did so in a way to meet quality standards for upstream inclusion (available [in mainline](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8d59598c35dc>) since Linux v4.15).

I later (re)discovered that the [almost 10-year-old AXFS](<https://lwn.net/Articles/295545/>) filesystem (still [maintained out of tree](<https://github.com/jaredeh/axfs>)) could have been a good fit. I had forgotten about it though, and in any case I prefer to work with mainline code.

One may wonder why [DAX](<https://lwn.net/Articles/717953/>) was not used here. DAX is like XIP on steroids; it is tailored for large writable filesystems and relies on the presence of an MMU (which the STM32 processor lacks) to page in and out data as needed. Its [documentation](<https://www.kernel.org/doc/Documentation/filesystems/dax.txt>) also mentions another shortcoming: ""The DAX code does not work correctly on architectures which have virtually mapped caches such as ARM, MIPS and SPARC"". Because cramfs with XIP is read-only and small enough to always be entirely mapped in memory, it is possible to achieve the intended result with a much simpler approach, making DAX somewhat overkill in this context.

#### User-space XIP and executable binary formats

Now that we're set with an XIP-capable filesystem, it is time to populate it. I'm using a static build of [BusyBox](<https://busybox.net/>) to keep things simple. Using a target with an MMU, we can see how things are mapped in memory:

```
# cat /proc/self/maps
        00010000-000a5000 r-xp **08101000** 1f:00 1328       /bin/busybox
        000b5000-000b7000 rw-p 00095000 1f:00 1328       /bin/busybox
        000b7000-000da000 rw-p 00000000 00:00 0          [heap]
        bea07000-bea28000 rw-p 00000000 00:00 0          [stack]
        bebc1000-bebc2000 r-xp 00000000 00:00 0          [sigpage]
        bebc2000-bebc3000 r--p 00000000 00:00 0          [vvar]
        bebc3000-bebc4000 r-xp 00000000 00:00 0          [vdso]
        ffff0000-ffff1000 r-xp 00000000 00:00 0          [vectors]
```

The clue that gives XIP away is shown in bold in the third column on the first output line. It is meant to be the file offset for that mapping, except that [`remap_pfn_range()`](<https://elixir.bootlin.com/linux/v4.15/source/mm/memory.c#L2035>), used to establish an XIP mapping, overwrites the file offset in the virtual memory area (VMA) structure (`vma->vm_pgoff`) with the physical address for that mapping. We can see that 0x08101000 would be way too big for a file offset here; instead, it corresponds to a location in the physical address range for the flash memory. Cramfs may also use [`vm_insert_mixed()`](<https://elixir.bootlin.com/linux/v4.15/source/mm/memory.c#L1933>) in some cases, and then this physical address reporting wouldn't be available. A reliable way to display XIP mappings in all cases would be useful.

The second `/bin/busybox` mapping (the `.data` section) is flagged read-write (`rw-p`), unlike the first one (the `.text` section) which is read-only and executable (`r-xp`). Writable segments cannot be mapped to the flash memory and, therefore, have to be loaded in RAM in the usual way.

The MMU makes it easy for a program to see its code at the absolute address it expects regardless of the actual memory used. Things aren't that simple in the no-MMU case, where user executables must be able to run at any memory address; [position-independent code (PIC)](<https://en.wikipedia.org/wiki/Position-independent_code>) is therefore a requirement. This ability is offered by the [bFLT flat file format](<http://www.uclinux.org/bFLT/>), and has been available for quite a long time with [uClinux](<http://www.uclinux.org/>) targets. However, this format has multiple limitations that make XIP, shared libraries, or the combination of both, unwieldy.

Fortunately there is a variant of [ELF](<https://en.wikipedia.org/wiki/Executable_and_Linkable_Format>), called [ELF FDPIC](<https://github.com/mickael-guene/fdpic_doc/blob/master/abi.txt>), that overcomes all those limitations. Because FDPIC segments are position-independent with no predetermined relative offset between them, it is possible to share common `.text` segments across multiple executable instances just like standard ELF-on-MMU targets, and those `.text` segments may be XIP as well. ELF FDPIC support was added to the ARM architecture (also available [in mainline](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=382e67aec6a7>) since Linux v4.15).

On my STM32 target, with the combination of a XIP-enabled cramfs and ELF FDPIC user-space binaries, the BusyBox mapping now looks like this:

```
# cat /proc/self/maps
        00028000-0002d000 rw-p 00037000 1f:03 1660       /bin/busybox
        0002d000-0002e000 rw-p 00000000 00:00 0
        0002e000-00030000 rw-p 00000000 00:00 0          [stack]
        081a0760-081d8760 r-xs 00000000 1f:03 1660       /bin/busybox
```

Due to the lack of an MMU, the XIP segment is even more obvious as there is no address translation and the flash-memory address is clearly visible. The [no-MMU memory mapping support](<https://www.kernel.org/doc/Documentation/nommu-mmap.txt>) requires shared mappings for XIP, hence the "`r-xs`" annotation.

#### Hammering static memory down

Okay, now we're all set for some hammering. We've seen that our XIP BusyBox above already saved 229,376 bytes of RAM, or 56 memory pages. That represents 44% of our total budget of 128 pages if we want to target 512KB of RAM. From now on, it is important to closely track where memory allocations go and determine how useful that precious memory is. Let's start by looking at the kernel itself, using a trimmed-down configuration from the previous article, but with `CONFIG_XIP_KERNEL=y` (and LTO disabled for now as it takes too long to build). We get:

```
text    data     bss     dec     hex filename
        1016264   97352  169568 1283184  139470 vmlinux
```

The 1,016,264 bytes of text are located in flash so we can ignore them for a while. The 266,920 bytes of data and BSS, though, represent 51% of our RAM budget. Let's find out what is responsible for it with some scripting on the `System.map` file:

```
#!/bin/sh
        {
            read addr1 type1 sym1
            while read addr2 type2 sym2; do
                size=$((0x$addr2 - 0x$addr1))
                case $type1 in
                b|B|d|D)
                    echo -e "$type1 $size\t$sym1"
                    ;;
                esac
                type1=$type2
                addr1=$addr2
                sym1=$sym2
            done
        } < System.map | sort -n -r -k 2
```

The first output lines are:

```
B 133953016     _end
        b 131072        __log_buf
        d 8192  safe_print_seq
        d 8192  nmi_print_seq
        D 8192  init_thread_union
        d 4288  timer_bases
        b 4100  in_lookup_hashtable
        b 4096  ucounts_hashtable
        d 3960  cpuhp_ap_states
        [...]
```

Here we ignore `_end` because its apparent huge size comes from the fact that the end of kernel static allocation in RAM comes before the next kernel symbol located in flash — much higher in the address space. It is always good to go back to `System.map` to make sense of some weird cases like this.

However, we do have a clearly identifiable memory allocation to pound on. Looking at [ the declaration of `__log_buf`](<https://elixir.bootlin.com/linux/v4.15/source/kernel/printk/printk.c#L432>) we see:

```
/* record buffer */
        #define LOG_ALIGN __alignof__(struct printk_log)
        #define __LOG_BUF_LEN (1 << CONFIG_LOG_BUF_SHIFT)
        static char __log_buf[__LOG_BUF_LEN] __aligned(LOG_ALIGN);
```

This one is easy. Because we don't want to configure the whole of `printk()` support out just yet, we'll set `CONFIG_LOG_BUF_SHIFT=12` (the smallest allowed value). And while there, we'll also set the configuration symbol `CONFIG_PRINTK_SAFE_LOG_BUF_SHIFT` to its minimum of ten. The result is:

```
text    data     bss     dec     hex filename
        1016220   83016   42624 1141860  116c64 vmlinux
```

Our RAM usage went down from 266,920 to 125,640 bytes with a couple of simple configuration tweaks. Let's see our symbol-size list again:

```
B 134092280     _end
        D 8192  init_thread_union
        d 4288  timer_bases
        b 4100  in_lookup_hashtable
        b 4096  ucounts_hashtable
        b 4096  __log_buf
        d 3960  cpuhp_ap_states
        [...]
```

The next contender is [ `init_thread_union`](<https://elixir.bootlin.com/linux/v4.15/source/init/init_task.c#L23>). This one is interesting because its size is derived from [ `THREAD_SIZE_ORDER`](<https://elixir.bootlin.com/linux/v4.15/source/arch/arm/include/asm/thread_info.h#L19>), which determines how many stack pages each kernel task gets. The first task (the init task) happens to have its stack statically allocated in the `.data` segment, which is why we see it here. Changing this from two pages to one page should be perfectly fine for our tiny environment, and this will also save one page per task with dynamically allocated stacks.

To reduce the size of [ `timer_bases`](<https://elixir.bootlin.com/linux/v4.15/source/kernel/time/timer.c#L197>) we'll tweak the value of [ `LVL_BITS`](<https://elixir.bootlin.com/linux/v4.15/source/kernel/time/timer.c#L165>) down from six to four. To reduce [ `in_lookup_hashtable`](<https://elixir.bootlin.com/linux/v4.15/source/fs/dcache.c#L118>) we change [ `IN_LOOKUP_SHIFT`](<https://elixir.bootlin.com/linux/v4.15/source/fs/dcache.c#L117>) from ten to five. And so on for a few more random kernel constants.

#### Nailing down dynamic memory allocations

Figuring out and reducing static memory allocations is easy as we've seen. But dynamic allocations must be dealt with as well, and for that we have to instrument our target and boot it. The first dynamic allocations come from the [memblock allocator](<https://elixir.bootlin.com/linux/v4.15/source/mm/memblock.c>), as the usual kernel memory allocators are not up and running yet. All the instrumentation we need is already there; it suffices to provide "`memblock=debug`" on the kernel command line to activate it. Here's what it shows:

```
memblock_reserve: [0x00008000-0x000229f7] arm_memblock_init+0xf/0x48
        memblock_reserve: [0x08004000-0x08007dbe] arm_memblock_init+0x1d/0x48
```

Here we have our static RAM being reserved, followed by our kernel code and read-only data in flash (which is mapped starting at 0x08004000). If the kernel code were in RAM then it would make sense to reserve that too. In this case this is just a useless but harmless reservation since the flash will never be allocated for any other purpose anyway.

Now for actual dynamic allocations:

```
memblock_virt_alloc_try_nid_nopanic: 131072 bytes align=0x0 nid=0
        from=0x0 max_addr=0x0 alloc_node_mem_map.constprop.6+0x35/0x5c
          Normal zone: 32 pages used for memmap
          Normal zone: 4096 pages, LIFO batch:0
```

This is our `memmap` array taking up 131,072 bytes (32 pages) in order to manage 4096 pages. By default this target uses the full 16MB of external RAM available on the board. So if we reduce the number of available pages, to say 512KB, then this will shrink significantly.

The next significant allocation is:

```
memblock_virt_alloc_try_nid_nopanic: 32768 bytes align=0x1000 nid=-1
        from=0xffffffff max_addr=0x0 setup_per_cpu_areas+0x21/0x64
        pcpu-alloc: s0 r0 d32768 u32768 alloc=1*32768
```

32KB of per-CPU memory pool for a uniprocessor system with less than a megabyte of RAM? Nah. Here are a few tweaks to [ `include/linux/percpu.h`](<https://elixir.bootlin.com/linux/v4.15/source/include/linux/percpu.h>) to reduce that to a single page:

```
-#define PCPU_MIN_UNIT_SIZE             PFN_ALIGN(32 << 10)
        +#define PCPU_MIN_UNIT_SIZE             PFN_ALIGN(4 << 10)
    
        -#define PERCPU_DYNAMIC_EARLY_SLOTS     128
        -#define PERCPU_DYNAMIC_EARLY_SIZE      (12 << 10)
        +#define PERCPU_DYNAMIC_EARLY_SLOTS     32
        +#define PERCPU_DYNAMIC_EARLY_SIZE      (4 << 10)
    
        +#undef PERCPU_DYNAMIC_RESERVE
        +#define PERCPU_DYNAMIC_RESERVE         (4 << 10)
```

It is worth noting that only the SLOB memory allocator (`CONFIG_SLOB`) still works after these changes.

Moving on to the next major allocation:

```
memblock_virt_alloc_try_nid_nopanic: 8192 bytes align=0x0 nid=-1
        from=0x0 max_addr=0x0 alloc_large_system_hash+0x119/0x1a4
        Dentry cache hash table entries: 2048 (order: 1, 8192 bytes)
        memblock_virt_alloc_try_nid_nopanic: 4096 bytes align=0x0 nid=-1
        from=0x0 max_addr=0x0 alloc_large_system_hash+0x119/0x1a4
        Inode-cache hash table entries: 1024 (order: 0, 4096 bytes)
```

Who said this is a _large_ system? Yes, you should get the idea by now; a couple more small tweaks are needed, but they're omitted from this article for the sake of keeping it reasonably short.

After that, the usual kernel memory allocators such as `kmalloc()` take over, and allocations ultimately end up down in [ `__alloc_pages_nodemask()`](<https://elixir.bootlin.com/linux/v4.15/source/mm/page_alloc.c#L4212>). The same kind of tracing and tweaks may be applied until the boot is complete. Sometimes it is just a matter of configuring out more stuff, such as the sysfs filesystem, whose memory needs are a bit excessive for our budget, and so on.

#### Back to user space

Now that we have hammered down the kernel's RAM usage, we're ready to flash and boot it again. The minimum amount of RAM required for a successful boot to user space at this point is 800KB ("`mem=800k`" on the kernel command line). Let's explore our small world:

```
BusyBox v1.7.1 (2017-09-16 02:45:01 EDT) hush - the humble shell
    
        # free
                     total       used       free     shared    buffers     cached
        Mem:           672        540        132          0          0          0
        -/+ buffers/cache:        540        132
    
        # cat /proc/maps
        00028000-0002d000 rw-p 00037000 1f:03 1660       /bin/busybox
        0002d000-0002e000 rw-p 00000000 00:00 0
        0002e000-00030000 rw-p 00000000 00:00 0
        00030000-00038000 rw-p 00000000 00:00 0
        0004d000-0004e000 rw-p 00000000 00:00 0
        00061000-00062000 rw-p 00000000 00:00 0
        0006c000-0006d000 rw-p 00000000 00:00 0
        0006f000-00070000 rw-p 00000000 00:00 0
        00070000-00078000 rw-p 00000000 00:00 0
        00078000-0007d000 rw-p 00037000 1f:03 1660       /bin/busybox
        081a0760-081d8760 r-xs 00000000 1f:03 1660       /bin/busybox
```

Here we can see two four-page RAM mappings from offset 0x37000 of `/bin/busybox`. Those are two data instances, one for the shell process, and one for the `cat` process. They both share the busybox XIP code segment at 0x081a0760, which is good. There are also two anonymous eight-page RAM mappings among much smaller ones though, and that eats our page budget pretty quickly. They correspond to a 32KB stack space for each of those processes. That certainly can be tweaked down:

```
--- a/fs/binfmt_elf_fdpic.c
        +++ b/fs/binfmt_elf_fdpic.c
        @@ -337,6 +337,7 @@ static int load_elf_fdpic_binary(struct linux_binprm *bprm)
                retval = -ENOEXEC;
                if (stack_size == 0)
                        stack_size = 131072UL; /* same as exec.c's default commit */
    
        +       stack_size = 8192;
    
                if (is_constdisp(&interp_params.hdr))
                        interp_params.flags |= ELF_FDPIC_FLAG_CONSTDISP;
```

That's certainly a quick and nasty hack; properly changing the stack size in the ELF binary's header is the way to go. It would also require careful validation, say on a MMU system with a fixed-size stack where any stack overflow could be caught. But hey, it wouldn't be our first hack at this point and that will do for now.

Still, before rebooting, let's explore some more:

```
# ps
          PID USER       VSZ STAT COMMAND
            1 0          300 S    {busybox} sh
            2 0            0 SW   [kthreadd]
            3 0
        ps invoked oom-killer: gfp_mask=0x14000c0(GFP_KERNEL),
        nodemask=(null), order=0, oom_score_adj=0
        [...]
        Out of memory: Kill process 19 (ps) score 5 or sacrifice child
```

The intervention of the out-of-memory killer was bound to happen at some point, of course. However the out-of-memory report also provided this piece of information from the buddy allocator:

```
Normal: 2*4kB (U) 3*8kB (U) 2*16kB (U) 2*32kB (UM)
                0*64kB 0*128kB 0*256kB = 128kB
```

The `ps` process tried to perform a memory allocation with `order=0` (a single 4KB page) and this failed despite having 128KB still available. Why is that? It turns out that the page allocator does not like performing normal memory allocations when there isn't at least a certain small amount of free memory available, as enforced by [ `zone_watermark_ok()`](<https://elixir.bootlin.com/linux/v4.15/source/mm/page_alloc.c#L2993>). This is to avoid possible deadlocks if a failed memory allocation results in the killing of a process — an operation that may require memory allocations of its own. Even though this watermark is supposed to be small, in our tiny environment this is still something we don't need and can't afford. So let's lower those watermarks slightly:

```
--- a/mm/page_alloc.c
        +++ b/mm/page_alloc.c
        @@ -7035,6 +7035,10 @@ static void __setup_per_zone_wmarks(void)
                        zone->watermark[WMARK_LOW]  = min_wmark_pages(zone) + tmp;
                        zone->watermark[WMARK_HIGH] = min_wmark_pages(zone) + tmp * 2;
    
        +               zone->watermark[WMARK_MIN] = 0;
        +               zone->watermark[WMARK_LOW] = 0;
        +               zone->watermark[WMARK_HIGH] = 0;
        +
                        spin_unlock_irqrestore(&zone->lock, flags);
                }
```

Finally we're able to reboot with "`mem=768k`" on the kernel command line:

```
Linux version 4.15.0-00008-gf90e37b6fb-dirty (nico@xanadu.home) (gcc version 6.3.1 20170404
     		  (Linaro GCC 6.3-2017.05)) #634 Fri Feb 23 14:03:34 EST 2018
        CPU: ARMv7-M [410fc241] revision 1 (ARMv7M), cr=00000000
        CPU: unknown data cache, unknown instruction cache
        OF: fdt: Machine model: STMicroelectronics STM32F469i-DISCO board
        On node 0 totalpages: 192
          Normal zone: 2 pages used for memmap
          Normal zone: 0 pages reserved
          Normal zone: 192 pages, LIFO batch:0
        random: fast init done
        [...]
    
        BusyBox v1.27.1 (2017-09-16 02:45:01 EDT) hush - the humble shell
    
        # free
                     total       used       free     shared    buffers     cached
        Mem:           644        532        112          0          0         24
        -/+ buffers/cache:        508        136
    
        # ps
          PID USER       VSZ STAT COMMAND
            1 0          276 S    {busybox} sh
            2 0            0 SW   [kthreadd]
            3 0            0 IW   [kworker/0:0]
            4 0            0 IW<  [kworker/0:0H]
            5 0            0 IW   [kworker/u2:0]
            6 0            0 IW<  [mm_percpu_wq]
            7 0            0 SW   [ksoftirqd/0]
            8 0            0 IW<  [writeback]
            9 0            0 IW<  [watchdogd]
           10 0            0 IW   [kworker/0:1]
           11 0            0 SW   [kswapd0]
           12 0            0 SW   [irq/31-40002800]
           13 0            0 SW   [irq/32-40004800]
           16 0            0 IW   [kworker/u2:1]
           21 0            0 IW   [kworker/u2:2]
           23 0          260 R    ps
    
        # grep -v " 0 kB" /proc/meminfo
        MemTotal:            644 kB
        MemFree:              92 kB
        MemAvailable:         92 kB
        Cached:               24 kB
        MmapCopy:             92 kB
        KernelStack:          64 kB
        CommitLimit:         320 kB
```

Here it is! Not exactly our target of 512KB of RAM but 768KB is getting pretty close. Some microcontrollers already have more than that amount of available on-chip SRAM.

Easy improvements are still possible. We can see above that 14 out of the 16 tasks are kernel threads, each with their 4KB stack; some of them could certainly go. Going through another round of memory page tracking would reveal yet more things that could be optimized out, etc. Yet, a dedicated application that doesn't spawn child processes is likely to require less RAM to run as well, unlike this generic shell environment. After all, some popular microcontrollers that are able to connect to the Internet have less total RAM than the remaining free RAM we have here.

#### Conclusion

There is at least one important lesson to be learned from the work on this project: shrinking the kernel's RAM usage is much easier than shrinking its code size. The code tends to be highly optimized already, because it has a direct influence on system performance, even on big systems. That is not necessarily the case for actual memory usage though. RAM comes relatively cheap on big systems, and wasting some of it really doesn't matter much in practice. Therefore, much low-hanging fruit can be found when optimizing RAM usage for small systems. 

Other than the small tweaks and quick hacks presented here, all the major pieces relied upon in this article (XIP kernel, XIP user space, even some [ device tree memory usage reduction](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=77ea8a68c54e>)) are available in the mainline already. But further work beyond this proof of concept is still needed to make Linux on tiny devices really useful. Progression of this work will depend, as always, on people's desire to use it and willingness to form a community to promote its development.

[Thanks to [Linaro](<https://www.linaro.org/>) for allowing me to work on this project and to write this article series.]
