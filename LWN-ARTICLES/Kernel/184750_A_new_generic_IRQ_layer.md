---
title: A new generic IRQ layer
url: https://lwn.net/Articles/184750/
date: "May 23, 2006"
category: Interrupts
author: 
---

The Linux kernel has a generic layer for the handling of hardware interrupts, hidden behind a standard API. There's only one problem: not all architectures use this layer. In particular, ARM is a holdout. It seems that interrupt handling in the ARM world is a complicated, subarchitecture-specific business which does not fit into the current "generic" code at all, so ARM sticks with its own code - even though there is a fair amount of overlap with code found in the generic subsystem. But, even for the architectures which are able to use it, the current IRQ subsystem has shortcomings which are becoming increasingly apparent. 

An attempt to change the situation can be seen in the [genirq patch set](<http://lwn.net/Articles/184408/>) by Thomas Gleixner and Ingo Molnar. These patches attempt to take lessons learned about optimal interrupt handling on all architectures, mix in the quirks found in the fifty (yes, fifty) ARM subarchitectures, and create a new IRQ subsystem which is truly generic, and more powerful as well. It is a big patch set which reworks a great deal of crucially important low-level code. Expect some interesting discussion before any eventual mainline merge. 

After some cleanup work, the patch gets serious with the creation of a new `irq_chip` structure. This structure is based on the old `hw_interrupt_type` structure, but it includes a rather longer list of low-level operations. The things for which the kernel can now request a specific interrupt controller include: 

  * `startup()`: enable the interrupt and generally get the controller ready to handle it. 
  * `shutdown()`: completely shut down the interrupt. 
  * `enable()`: enable the interrupt. 
  * `disable()`: disable the interrupt. 
  * `ack()`: inform the controller that the CPU has begun processing the interrupt. 
  * `end()`: inform the controller that interrupt processing is done. 
  * `mask()`: mask a specific interrupt, blocking its delivery. 
  * `mask_ack()`: a combination of `mask()` and `ack()` which can be optimized on some platforms. 
  * `unmask()`: unmask an interrupt. 
  * `set_affinity()`: bind an interrupt to a specific CPU. 
  * `retrigger()`: re-create and re-deliver an interrupt. 
  * `set_type()`: set the flow type (described below) of the interrupt. 
  * `set_wake()`: enable or disable wake-on-interrupt behavior. 

Many of these methods existed previously, but the `mask()`, `mask_ack()`, `unmask()`, `set_type()`, and `set_wake()` functions are new. With this set of functions, kernel code can manage interrupt controller chips in a fine-grained manner. 

Moving up a level, the existing `irq_desc` structure, which holds all of the kernel's information about any specific interrupt, now has a pointer to an associated `irq_chip` structure. It also has a new method, `handle_irq()`, pointing to the function which actually handles this interrupt. That, perhaps, is the most fundamental change from the existing system, which uses a single handler function (`__do_IRQ()`) for all interrupts. It is a recognition of the fact that not all interrupts are equal, so there is little to gain by trying to deal with them all in a single, big function. 

The biggest difference between interrupts is what is called the "flow type" - a combination of how the interrupt is signaled and how the system processes it. The genirq patches define these flow types: 

  * Level-triggered interrupts are active as long as the device asserts its IRQ line. These interrupts must be masked while being processed, and can only be unmasked after the device has stopped asserting the interrupt. 

  * Edge-triggered interrupts are signaled by a change in the interrupt line - from low voltage to high, from high to low, or both. These interrupts do not necessarily have to be masked while being processed, but, if they are not masked, more interrupts can arrive before the first has been handled. So the kernel must track "pending" interrupts, and the interrupt handler must loop until all interrupts have been dealt with. 

  * "Simple" interrupts do not require any special control, and can be processed directly. 

  * Per-CPU interrupts are bound to a single CPU. They are much like simple interrupts, but even simpler: since the handler will only run on one CPU, there is no need for locking. 

The current IRQ code attempts to handle all of the above cases in a single, large routine. The new code, instead, creates a number of flow-specific handler functions, then sets the appropriate one as the `handle_irq()` method in the interrupt descriptor. The result is code which can be optimized for specific needs, and shorter code paths in the interrupt system as a whole. If a particular hardware platform has quirks which are not addressed by the current handlers, creating a new one is a relatively straightforward task. 

At the kernel API level, the changes are relatively small; changes to drivers are not generally required. There are a few new capabilities, however. One is that there are some new flags which can be passed to `request_irq()`: 

  * `SA_TRIGGER_LOW` and `SA_TRIGGER_HIGH`: treat the interrupt source as being level-triggered, with interrupts happening at either the high or low level. 

  * `SA_TRIGGER_FALLING` and `SA_TRIGGER_RISING`: treat the interrupt as being edge-triggered. 

This addition to the API actually happened in 2.6.16, but only the ARM architecture had any support for it at all. With the genirq patches, all architectures support these flags, and the appropriate flow handler will be selected internally. When interrupts are shared, however, all users must agree on how the triggering will be handled. 

It is also possible to change the flow type of an IRQ directly with: 

```
int set_irq_type(unsigned int irq, unsigned int type);
```

Here, `type` should be one of `IRQ_TYPE_EDGE_RISING`, `IRQ_TYPE_EDGE_FALLING`, `IRQ_TYPE_EDGE_BOTH`, `IRQ_TYPE_LEVEL_HIGH`, `IRQ_TYPE_LEVEL_LOW`, `IRQ_TYPE_SIMPLE`, or `IRQ_TYPE_PERCPU`. Calling this function has the same effect as specifying the trigger type with `request_irq()`, but it offers a wider range of possibilities. It also does not check for compatibility with any other users of a shared interrupt, so a certain potential for confusion exists. 

Some devices can generate interrupts which should wake up the system from a suspended state. Wake-on-LAN behavior in network adaptors is one example; allowing the keyboard to wake the system is another. Kernel code can enable or disable this behavior in the interrupt controller with: 

```
int set_irq_wake(unsigned int irq, unsigned int on);
```

An error code will be returned if the chip-level controller does not implement this operation. 

There has been a relatively small amount of discussion so far; the biggest [objection](<https://lwn.net/Articles/184778/>) seems to be a claim that the separate flow handlers are an unnecessarily complex addition. The decision on whether genirq is merged very likely depends on whether the ARM maintainers are willing to drop their architecture-specific IRQ implementation and move to the new, generic version. Without that, the genirq code, which contains a lot of work aimed specifically at ARM's needs, will not truly be a generic solution. In the mean time, genirq has found its way into the -mm tree.
