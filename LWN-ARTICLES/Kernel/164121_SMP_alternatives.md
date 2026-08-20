---
title: SMP alternatives
url: https://lwn.net/Articles/164121/
date: "December 14, 2005"
category: Alternative instructions; Virtualization
author: 
---

The i386 processor family poses a challenge for kernel builders. These processors have maintained instruction set compatibility for many years; code built for early Pentium processors will likely still run on current hardware. The problem is that code built for these older processors will fail to take advantage of features added later on. The "least common denominator" approach can thus lead to sub-optimal use of current CPUs. 

The kernel has a number of ways of dealing with this challenge. In some cases it can make decisions at run time, using processor features only if they are found to be present. Other features are only available by way of build-time configuration options; selecting these will result in a kernel which will not run on older systems. Yet another mechanism is the "alternatives" feature, which allows the kernel to optimize itself at boot time. Consider this example of alternatives use (from `include/asm-i386/system.h`): 

```
#define mb() alternative("lock; addl $0,0(%%esp)", \
                                 "mfence", \
    			     X86_FEATURE_XMM2)
```

This macro places a memory barrier in the code, ensuring that all memory reads and writes initiated before the barrier complete before execution continues. The default implementation is essentially a bus-locked no-op; it will work anywhere. On newer systems, however, the more efficient `mfence` instruction is available, and it would be nice to use it. 

The `alternative()` macro compiles in the default code, but also makes a note of its location (and alternative implementation) in a special ELF section. Early in the boot process, the kernel calls `apply_alternatives()`, which makes a pass through that special section. Every alternative instruction which is supported by the running processor is patched directly into the loaded kernel image; it will be filled with no-op instructions if need be. Once `apply_alternatives()` has finished its work, the kernel behaves as if it had been compiled for the processor it is actually running on. This mechanism allows distributors to ship generic kernels which can optimize themselves at boot time. 

The 2.6 mainline uses alternatives sparingly: for barriers, prefetch hints, and saving the floating point unit state. Gerd Knorr, however, believes that the use of alternatives could be expanded to further reduce the range of kernels which distributors need to ship - and to improve runtime flexibility as well. In particular, he thinks that kernels can be optimized for single- or multiprocessor systems on the fly. 

Gerd's [SMP alternatives patch](<http://lwn.net/Articles/163810/>) is an implementation of this concept. It creates an new macro (`alternative_smp()`) which can be used to specify optimal implementations of an operation on both uniprocessor and SMP systems; the proper version will then be selected at runtime. The main use of SMP alternatives in his patch is with spinlock operations; spinlocks can be patched in or edited out, as dictated by the configuration of the system at boot time. 

There are a couple of interesting features in Gerd's patch. One is in the handling of the i386 architecture's `lock` prefix. This prefix, when applied to specific instructions, causes the instruction to run in a bus-locked, atomic manner. It is used for operations which must be seen coherently across a multiprocessor system; these include semaphore operations and the `atomic_t` implementation. Use of the `lock` prefix on uniprocessor systems imposes a runtime cost with no benefit; it would be nice to edit those out. The SMP alternatives patch takes a shortcut here; it simply remembers each location where a `lock` prefix appears. If the kernel boots on a uniprocessor system, all of those prefixes can be quickly overwritten with no-ops. 

A more interesting - and more controversial - feature of this patch is that, when the kernel is converted between the SMP and uniprocessor mode, the overwritten instructions are remembered. At some point the the future, then, the alternatives code can reverse the change, switching the kernel back to the full SMP implementation. The code is then run whenever a CPU hotplug event happens, optimizing the kernel for the system's new configuration. A system can be initially booted with a single processor, and the alternatives code will edit out all of the SMP-related instructions. If another processor is added later on, the kernel will be automatically converted back into a fully SMP-capable mode. If processors are removed, the SMP code can be taken out too. All within a running system, with no need to reboot. 

This feature may seem useful to a rather small minority of users - and it is. But that minority may be bigger than one thinks. Virtualization systems (and Xen in particular) are implementing the ability to configure the number of (virtual) CPUs in each running instance on the fly, in response to the load on each. So it may really be that a busy, virtualized server will have CPUs hot-plugged into it, and that those processors will go away when the load drops. Enabling the kernel to reconfigure itself on the fly when this happens will allow each Xen instance to run a kernel which is optimized for its current situation. 

The CPU hotplug may be a hard sell - self-modifying code in a running kernel tends to make people nervous. The rest of the SMP alternatives patch seems likely to find a place in the mainline, eventually.
