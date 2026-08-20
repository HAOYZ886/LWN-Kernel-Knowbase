---
title: Moving interrupts to threads
url: https://lwn.net/Articles/302043/
date: "October 8, 2008"
category: Interrupts
author: "By Jake Edge October 8, 2008"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
October 8, 2008

Processing interrupts from the hardware is a major source of latency in the kernel, because other interrupts are blocked while doing that processing. For this reason, the realtime tree has a feature, called _threaded interrupt handlers_ , that seeks to reduce the time spent with interrupts disabled to a bare minimum--pushing the rest of the processing out into kernel threads. But it is not just realtime kernels that are interested in lower latencies, so threaded handlers are being proposed for addition to the mainline. 

Reducing latency in the kernel is one of the benefits, but there are other advantages as well. The biggest is probably reducing complexity by simplifying or avoiding locking between the "hard" and "soft" parts of interrupt handling. Threaded handlers will also help the debuggability of the kernel and may eventually lead to the [removal of tasklets](<
http://lwn.net/Articles/239633/>) from Linux. For these reasons, and a few others as well, Thomas Gleixner has [posted](<http://lwn.net/Articles/301890/>) a set of patches and a "request for comments" to add threaded interrupt handlers. 

Traditionally, interrupt handling has been done with _top half_ (i.e. the "hard" irq) that actually responds to the hardware interrupt and a _bottom half_ (or "soft" irq) that is scheduled by the top half to do additional processing. The top half executes with interrupts disabled, so it is imperative that it do as little as possible to keep the system responsive. Threaded interrupt handlers reduce that work even further, so the top half would consist of a "quick check handler" that just ensures the interrupt is from the device; if so, it simply acknowledges the interrupt to the hardware and tells the kernel to wake the interrupt handler thread. 

In the realtime tree, nearly all drivers were mass converted to use threads, but the patch Gleixner proposes makes it optional--driver maintainers can switch if they wish to. Automatically converting drivers is not necessarily popular with all maintainers, but it has an additional downside as Gleixner notes: ""Converting an interrupt to threaded makes only sense when the handler code takes advantage of it by integrating tasklet/softirq functionality and simplifying the locking"". 

A driver that wishes to request a threaded interrupt handler will use: 

```
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
    	    		     irq_handler_t quick_check_handler,
    			     unsigned long flags, const char *name, void *dev)
```

This is essentially the same as `request_irq()` with the addition of the `quick_check_handler`. As [requested by Linus Torvalds](<http://lwn.net/Articles/298840/>) at this year's Kernel Summit, a new function was introduced rather than changing countless drivers to use a new `request_irq()`. 

The `quick_check_handler` checks to see if the interrupt was from the device, returning `IRQ_NONE` if it isn't. It can also return `IRQ_HANDLED` if no further processing is required or `IRQ_WAKE_THREAD` to wake the handler thread. One other return code was added to simplify converting to a threaded handler. A `quick_check_handler` can be developed prior to the `handler` being converted; in that case, it returns `IRQ_NEEDS_HANDLING` (instead of `IRQ_WAKE_THREAD`) which will call the handler in the usual way. 

`request_threaded_irq()` will create a thread for the interrupt and put a pointer to it in the `struct irqaction`. In addition, a pointer to the `struct irqaction` has been added to the `task_struct` so that handlers can check the `action` flags for newly arrived interrupts. That reference is also used to prevent thread crashes from causing an oops. One of the few complaints seen so far about the proposal was a [concern about wasting four or eight bytes](<https://lwn.net/Articles/302244/>) in each `task_struct` that was not an interrupt handler (i.e. the vast majority). That structure could be split into two types, one for the kernel and one for user space, but it is unclear whether that will be necessary. 

Andi Kleen has a more general [concern](<https://lwn.net/Articles/302245/>) that threaded interrupt handlers will lead to bad code: ""to be honest my opinion is that it will encourage badly written interrupt code longer term"", but he seems to be in the minority. There were relatively few comments, but most seemed in favor--perhaps many are waiting to see the converted driver as Gleixner promises to deliver "real soon". If major obstacles don't materialize, one would guess the `linux-next` tree would be a logical next step, possibly followed by mainline merging for 2.6.29.
