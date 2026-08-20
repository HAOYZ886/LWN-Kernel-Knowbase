---
title: Handling interrupts in user space
url: https://lwn.net/Articles/127698/
date: "March 15, 2005"
category: "Device drivers-In user space; Interrupts"
author: 
---

Peter Chubb has long been working on a project to move device drivers into user space. Getting drivers out of the kernel, he points out, would have a number of benefits. Faults in drivers (the source of a large percentage of kernel bugs) would be less likely to destabilize the entire system. Drivers could be easily restarted and upgraded. And a user-space implementation would make it possible to provide a relatively stable driver API, which would appeal to many vendors. 

Much of the support needed for user-space drivers is already in place. A process can communicate with hardware by mapping the relevant I/O memory directly into its address space, for example; that is how the X server works with video adaptors. One piece, however, is missing: user-space drivers cannot handle device interrupts. In many cases, a proper driver cannot be written without using interrupts, so a user-space implementation is not possible. 

Peter has now posted [his user-space interrupts patch](<https://lwn.net/Articles/127293/>) for review and possible inclusion. The mechanism that he ended up with is simple and easy to work with, but it suffers from an important limitation. 

The mechanism is this: a process wishing to respond to interrupts opens a new `/proc` file; for IRQ 10, the file would be `/proc/irq/10/irq`. A read on that file will yield the number of interrupts which have occurred since the last read. If no interrupts have occurred, the `read()` call will block until the next interrupt happens. The `select()` and `poll()` system calls are properly supported, so it is possible to include interrupt handling as just another thing to do in an event loop. 

On the kernel side, the real interrupt handler looks like this: 

```
static irqreturn_t irq_proc_irq_handler(int irq, void *vidp, 
                                                struct pt_regs *regs)
        {
     	struct irq_proc *idp = (struct irq_proc *)vidp;
     
     	BUG_ON(idp->irq != irq);
     	disable_irq_nosync(irq);
     	atomic_inc(&idp->count);
     	wake_up(&idp->q);
     	return IRQ_HANDLED;
        }
```

In other words, all it does is count the interrupt and wake up any process that might be waiting to handle it. 

The handler also disables the interrupt before returning. There is an important reason for this action: since the handler knows nothing of the device which is actually interrupting, it is unable to acknowledge or turn off the interrupt. So, when the handler returns, the device will still be signalling an interrupt. If the interrupt were not disabled in the processor (or the APIC), the processor would be interrupted (and the handler called) all over again, repeatedly - at least, when level-triggered interrupts are in use. Disabling the interrupt allows life to go on until the user-space process gets scheduled and is able to tend to the interrupting device. 

There is a problem here, however: interrupt lines are often shared between devices. Disabling a shared interrupt shuts it off for all devices using that line, not just the one being handled by a user-space driver. It is entirely possible that masking that interrupt will block a device which is needed by the user-space handler - a disk controller, perhaps. In that case, the system may well deadlock. For this reason, the patch does not allow user-space drivers to work with shared interrupts. This restriction avoids problems, but it also reduces the utility of the whole thing. 

One possible solution was [posted](<https://lwn.net/Articles/127714/>) by Alan Cox. He would require user-space processes to pass a small structure into the kernel describing the hardware's IRQ interface. It would be just enough for the kernel to tell if a particular device is interrupting, acknowledge that interrupt, and tell the device to shut up. With that in place, the kernel could let user space deal with what the device really needs while leaving the interrupt enabled. It has been [pointed out](<https://lwn.net/Articles/127715/>) that this simple scheme would not work with some of the more complicated hardware, but it would be a step in the right direction regardless. 

Meanwhile, Michael Raymond [described](<https://lwn.net/Articles/127716/>) a different user-space interrupt implementation (called "User Level Interrupt" or ULI) done at SGI. This patch is significantly more complicated. In this scheme, a user-space driver would register an interrupt handler function directly with the kernel. When an interrupt happens, the ULI code performs some assembly-code black magic so that its "return from interrupt" instruction jumps directly into the user-space handler, in user mode. Once that handler returns, the ULI library writes a code to a magic device which causes the kernel stack and related data structures to be restored to their pre-interrupt state. The implementation is more complex, and it currently only works on the ia-64 architecture, but it could conceivably offer better performance than the `/proc` method.
