---
title: "Memory-mapped I/O without mysterious macros"
url: https://lwn.net/Articles/780710/
date: "February 26, 2019"
category: mmiowb
author: "By Jonathan Corbet February 26, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 26, 2019

Concurrency is hard even when the hardware's behavior is entirely deterministic; it gets harder in situations where operations can be reordered in seemingly random ways. In these cases, developers tend to reach for barriers as a way of enforcing ordering, but explicit barriers are tricky to use and are often not the best way to think about the problem. It is thus common to see explicit barriers removed as code matures. That now seems to be happening with an especially obscure type of barrier used with memory-mapped I/O (MMIO) operations. 

The core idea behind MMIO is that a peripheral device makes a set of registers available on the system's memory bus. The kernel can map those registers into its address space, then control the device by reading and writing those registers. I/O buses have often taken liberties with ordering when it comes to delivering operations to peripherals; that leads to rituals like performing an unneeded read from a PCI device's register space to force previous writes to be posted. But in some cases, the hardware can take things further and reorder operations arriving from different CPUs, even if the code performing those operations is strictly ordered. That, of course, can lead to a variety of amusing mixups. 

Fortunately, kernel developers tend _not_ to be amused by such things, so they take steps to ensure that this type of reordering does not happen. Back in 2004, Jesse Barnes [introduced a special sort of memory barrier operation](<https://lwn.net/Articles/103183/>) called `mmiowb()`; its job was to ensure that all MMIO writes initiated prior to the barrier would be posted to the device before any writes initiated after the barrier. `mmiowb()` was duly adopted by developers whose code needs to run on the affected hardware; there are now several hundred call sites in the kernel. 

Explicit barrier operations can work, but they have their pitfalls. Developers must remember to insert barrier operations anywhere that reordering could cause things to go astray. That tempts developers to sprinkle them throughout the code without necessarily thinking about exactly what those barriers are protecting against. In normal use, barriers need to come in groups of two or more, one in each place where a race might happen, but there is no indication of where any given barrier's siblings have been placed in the code, making it harder to understand what is going on. Code relying on explicit barriers, as a result, can be subject to rare failures that are nearly impossible to reproduce or diagnose. 

Will Deacon would like to improve this situation. He recently posted [a patch set](<https://lwn.net/ml/linux-kernel/20190222185026.10973-1-will.deacon@arm.com/>) wherein `mmiowb()` is said to stand for "Mysterious Macro Intended to Obscure Weird Behaviors"; his objective is to remove this macro, or at least hide it from the view of most developers. The core idea is one that comes up often in software development: developers should not be counted on to get complex concurrency issues right, especially in situations where the computer can do it for them. 

MMIO registers must be protected from concurrent accesses by multiple CPUs in the system; if that hasn't been done, there is nothing that barriers can do to stave off disaster. In the kernel, the implication is that code performing MMIO must be holding a spinlock that will prevent other processors from getting in the way. Spinlocks have already been defined as barriers when it comes to access to system RAM, freeing most kernel code from having to worry about memory-ordering issues in spinlock-protected code. Deacon's plan is to extend this definition to MMIO ordering on systems that need it. 

The patch set creates a per-CPU array of structures like: 

```
struct mmiowb_state {
    	u16	nesting_count;
    	u16	mmiowb_pending;
        };
```

Then, three sets of hooks are placed in the low-level spinlock and MMIO code. The functions that acquire spinlocks will call this function after the acquisition succeeds: 

```
static inline void mmiowb_spin_lock(void)
        {
    	if (__this_cpu_inc_return(__mmiowb_state.nesting_count) == 1)
    	    __this_cpu_write(__mmiowb_state.mmiowb_pending, 0);
        }
```

This function increments `nesting_count`, which is essentially keeping track of the number of spinlocks currently held by each CPU. When the first lock is acquired (when `nesting_count` is incremented to one), the `mmiowb_pending` flag is set to zero, indicating that no MMIO write operations have (yet) been performed in this critical section. 

While I/O memory looks like memory, and it can be tempting to access it by simply dereferencing a pointer, that does not always work on every architecture, so kernel developers use helper functions instead. Deacon's patch set adds a call to `mmiowb_set_pending()` to the helpers that perform write operations; it simply sets the `mmiowb_pending` flag to one, indicating that MMIO write operations have been performed since the last time it was cleared. 

Finally, operations that release a spinlock will call: 

```
static inline void mmiowb_spin_unlock(void)
        {
    	if (__this_cpu_xchg(__mmiowb_state.mmiowb_pending, 0))
    	    mmiowb();
    	__this_cpu_dec_return(__mmiowb_state.nesting_count);
        }
```

Here, `mmiowb_pending` is set back to zero and simultaneously tested; if its previous value was non-zero, `mmiowb()` is called. Then the nesting count is decremented. 

With these changes in place, there is no longer any need for driver-level code to make explicit calls to `mmiowb()`. That will, instead, happen automatically whenever MMIO operations have been performed inside a spinlock-protected critical section. That frees driver authors from the need to think about whether MMIO barriers are needed in any given situation. It also ensures that code will do the right thing, even if it is written by a developer who tests on machines with stricter MMIO-ordering guarantees and who has never even heard of `mmiowb()`. 

It is thus unsurprising that nobody has spoken out against these changes, even though the patch set modifies 178 files. Linus Torvalds [said](<https://lwn.net/ml/linux-kernel/CAHk-=wgamK0=rNsCfDfDzmNXUF_MqUHb0okzqN1Tir9vm65pNg@mail.gmail.com/>) ""I love removing mmiowb()""; he did, however, have some comments on how to make the implementation a bit more efficient. A revision of the patch set can thus be expected; after that, chances are that `mmiowb()` calls (and MMIO-barrier-related weird behavior) in driver code will soon be a thing of the past.
