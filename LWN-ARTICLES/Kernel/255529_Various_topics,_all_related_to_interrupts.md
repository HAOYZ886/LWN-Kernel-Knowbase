---
title: "Various topics, all related to interrupts"
url: https://lwn.net/Articles/255529/
date: "October 24, 2007"
category: Interrupts; Race conditions
author: "By Jonathan Corbet October 24, 2007"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 24, 2007

An interrupt handler is the portion of a device driver which is charged with responding to interrupts from the hardware; at a minimum it should shut the hardware up and initiate any processing which needs to be performed. When your editor worked on the second edition of _Linux Device Drivers_ , the prototype for interrupt handlers looked like this: 

```
void handler(int irq, void *dev_id, struct pt_regs *regs);
```

The kernel development process is not particularly kind to book authors who, as a rule, prefer to see the ink dry on their creations before the text becomes obsolete. True to form, the handler prototype has changed a couple of times since LDD2, with the result that the 2.6.23 version looks like: 

```
irqreturn_t handler(int irq, void *dev_id);
```

Along the way, interrupt handlers gained a return type (used to tell the kernel whether an interrupt was actually processed or not) and lost the processor registers argument. One would think that this interface (along with those who attempt to document it) had suffered enough, but, it seems, there will be no rest in the near future. 

In particular, Jeff Garzik has [proposed that the `irq` argument be removed](<http://lwn.net/Articles/255414/>) from the interrupt handler prototype. There are very few interrupt handlers which actually use that argument currently. And, as it turns out, most of the remaining handlers do not actually need it; they are often using the interrupt number to identify the interrupting device, but the `dev_id` pointer already exists for just that purpose. Still, getting this patch into the kernel would require a significant amount of work, since every in-tree interrupt handler will have to be audited and fixed up. 

So Jeff is taking it slowly; this is not a patch set which is aimed at being merged for 2.6.24. Before it goes in, there is room for a lot of useful work cleaning up the current use of the `irq` argument in drivers, all of which would ease the eventual transition to the new call. Handlers which really need the IRQ number can call the new `get_irqfunc_irq()` function. But, [says Jeff](<https://lwn.net/Articles/255538/>), ""I am finding a ton of bugs in each get_irqfunc_irq() driver, so I would rather patiently sift through them, and push fixes and cleanups upstream."" Quite a few interrupt handler fixes resulting from this work have already been posted. 

Eric Biederman worries that converting all of the drivers could be a challenge; he has [posted a proposal](<https://lwn.net/Articles/255539/>) which would create two different interrupt registration and handler interfaces, allowing drivers which really need the IRQ number to continue to receive it. Jeff is confident that the extra structure will not be necessary, though. Thomas Gleixner, instead, [would like to see the patches merged immediately](<https://lwn.net/Articles/255541/>), but it is almost certain that this patch set will be given one more development cycle to mature before going into the mainline. 

Alexey Dobriyan, meanwhile, would like to fix up the interrupt-safe spinlock interface. Most code which requires a spinlock in the presence of interrupts calls: 

```
void spin_lock_irqsave(spinlock_t *lock, unsigned long flags);
```

The `flags` variable is used by the (architecture-specific) code to save any interrupt state which may be needed when `spin_unlock_irqrestore()` is called. The problem with this interface is that it is not particularly type-safe. Developers have been known to use an `int` type instead of `unsigned long`; that usage will generate no errors and it will work fine on the x86 architecture. It will, however, fail in ugly ways on some other architectures. 

So Alexey would like to turn `flags` into a new type (`irq_flags_t`). This type would initially be defined to be `unsigned long`, so the change would not break compilation. It would be annotated, though, so that the `sparse` utility could point out all of the places where `spin_lock_irqsave()` is called with an incorrect type. In the more distant future, when the changeover is complete, architecture maintainers would be able to redefine the type to whatever works best on their systems, be it a structure or a single byte. 

Andrew Morton had [a mixed response](<https://lwn.net/Articles/255547/>) to the patch: 

Yes, it's always been ugly that we use unsigned long for this rather than abstracting it properly. 

However I'd prefer that we have some really good reason for introducing irq_flags_t now. Simply so that I don't needlessly spend the next two years wrestling with literally thousands of convert-to-irq_flags_t patches and having to type "please use irq_flags_t here" in hundreds of patch reviews. 

As an alternative, it was suggested that most calls of `spin_lock_irqsave()` should be changed to `spin_lock_irq()` instead. The latter version disables interrupts without saving the previous state; the accompanying `spin_unlock_irq()` call will then unconditionally re-enable interrupts. Those functions can be made to work, but only if it is known that interrupts will not have already been disabled when `spin_lock_irq()` is called. Otherwise the `spin_unlock_irq()` call risks enabling interrupts when some other part of the kernel expects them to still be disabled. The resulting random behavior is generally seen as undesirable by most computer users. So, in other words, `spin_lock_irqsave()` is a safer interface, which is why there is not a great deal of support for removing it. The prospect of well-intentioned kernel janitors changing code to `spin_lock_irq()` without really understanding the broader context is just too scary. 

Finally, there was a discussion involving `synchronize_irq()` which illustrates just how hard it can be to get a handle on race conditions on multiprocessor systems. This function: 

```
void synchronize_irq(unsigned int irq);
```

is intended to help coordinate actions between a driver's interrupt and non-interrupt code. At its core, it is a simple loop: 

```
while (desc->status & IRQ_INPROGRESS)
    	cpu_relax();
```

In other words, `synchronize_irq()` will busy-wait until it is known that no handlers are running for the given interrupt. The idea is that any interrupt handler which might have been running before the call to `synchronize_irq()` will have completed when that function returns. The typical usage pattern is something like this: 

```
some_important_flag = a_new_value;
        synchronize_irq();
        /* Code which depends on IRQ handler seeing a_new_value here */
```

With code like this, after the `synchronize_irq()` call, any interrupt handler will be guaranteed to see `a_new_value` \- or so people think. 

The problem is that contemporary processors will happily reorder memory operations to avoid pipeline stalls and improve performance; the [what every programmer should know about memory series](<http://lwn.net/Articles/250967/>) currently being serialized by LWN describes these issues in detail. What is relevant here is that the change to `some_important_flag` might be reordered (delayed) such that it does not become visible to other processors on the system until sometime after `synchronize_irq()` returns. During the window when the change is not visible, the promise of `synchronize_irq()` is not kept - an interrupt handler could run and see the old value, possibly creating mayhem as a result. That is the sort of obscure, one-in-a-billion race condition which keeps kernel hackers up at night. 

Actually, kernel hacking and coffee keep kernel hackers up at night, but your editor's point should be clear. 

Benjamin Herrenschmidt, upon finding this race, [attempted to fix it with a memory barrier](<https://lwn.net/Articles/255552/>). After some discussion, though, it became clear that the memory barrier was not sufficient. Barriers can affect the order in which operations become visible, but they cannot, in the absence of corresponding barriers on another processor, guarantee that a specific change becomes visible to that processor at any given time. That sort of guarantee requires the use of a locked operation which forces synchronization between processors - the sort of operation which is typically used to implement spinlocks. 

So the real solution appears to be [this patch](<https://lwn.net/Articles/255553/>) by Linus Torvalds and Herbert Xu. The `while` loop shown above persists in the new version, and it continues to run with no locks held - holding the interrupt descriptor lock when the interrupt subsystem may want it could lead to deadlocks. But, once it appears that no handlers are running, the descriptor lock is acquired and the status is checked one more time. If no handlers are running, the synchronize operation is complete; otherwise the code goes back to busy-waiting. The acquisition of the descriptor lock guarantees that memory barriers will have been executed on both sides of any potential race condition; that, in turn, will force the ordering of the memory operations. So, with this change in place, `synchronize_irq()` will truly synchronize with IRQ handlers and one more difficult race condition will have been eliminated.
