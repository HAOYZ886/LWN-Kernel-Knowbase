---
title: Improving lost and spurious IRQ handling
url: https://lwn.net/Articles/392136/
date: "June 15, 2010"
category: Interrupts
author: "By Jonathan Corbet June 15, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 15, 2010

Interrupts are a device's way of telling the kernel that something interesting has happened. One of the key benefits of using interrupts is that they free the kernel from the need to poll a device to learn what its state is. Like any other part of a computer, though, interrupts can go wrong, leading to situations where the system is overwhelmed by a flood of spurious interrupts - or, instead, left waiting for an interrupt which will never arrive. The kernel has some defensive mechanisms in its generic interrupt layer for dealing with situations like these; Tejun Heo has now posted [a patch series](<http://lwn.net/Articles/391973/>) intended to improve those mechanisms. As it happens, the necessary response when interrupts go bad is returning to polling. 

One problem which is familiar to driver authors is missing interrupts. A driver will typically set up an I/O operation, get it started, then wait until an interrupt indicating completion arrives. If that interrupt never shows up, the driver can end up waiting for a very long time. Missing interrupts can have a number of causes, including flaky devices or an interrupt routing problem somewhere in the system. Either way, if the driver author has not anticipated this situation and taken the appropriate measures - setting a timeout, for example - things will not end well. 

Waiting for interrupt timeouts will slow a device's performance considerably, though. That problem can be mitigated by polling the device state frequently, but rapid polling has its own costs. In an attempt to obtain the best results consistently, Tejun's patch adds a new driver API: 

```
#include <linux/interrupt.h>
    
        struct irq_expect *init_irq_expect(unsigned int irq, void *dev_id);
        void expect_irq(struct irq_expect *exp);
        void unexpect_irq(struct irq_expect *exp, bool timedout);
```

A call to `init_irq_expect()` will allocate an opaque token to be used with the other two functions; it should be passed the interrupt number of interest and the same `dev_id` value as was used to allocate the interrupt initially. When the driver initiates an action which should result in a device interrupt, it should make a call to `expect_irq()`. When the operation is completed, `unexpect_irq()` should be called, with `timedout` indicating whether the operation timed out (the interrupt did not arrive). Note that it's not necessary for the driver to free the `struct irq_expect` structure; that will happen automatically when the interrupt is released. 

A call to `expect_irq()` will initiate polling on the given interrupt line, where "polling" means making an occasional call to the device's interrupt handler. Initially, that polling is quite slow. If it turns out that the device is dropping interrupts (as indicated by the `timedout` parameter to `unexpect_irq()`), the polling frequency will be increased - up to once every millisecond. Working devices should interrupt before the slow poll period passes, so the result should be no real polling at all on reliable devices. If there is a problem with interrupt delivery, though, the kernel will automatically take responsibility for poking the interrupt handler when interrupts are expected. 

This interface works well if the driver knows when to expect interrupts, but not all devices work that way. For hardware which can interrupt at any time, there is an "IRQ watching" API instead: 

```
void watch_irq(unsigned int irq, void *dev_id);
```

This function will begin polling of the specified interrupt line; it will also initiate tracking of interrupt delivery status. If it determines that interrupts are being lost (as determined by an `IRQ_HANDLED` return status from a polled call to the handler), it will continue to poll at a higher frequency. Otherwise, eventually, interrupt delivery will be deemed to be reliable and polling will be turned off. 

Tejun's patch also changes the way that the kernel responds to spurious interrupts - those which no driver is interested in. Current kernels count the number of interrupts on each line for which no handler returned `IRQ_HANDLED`; if 99,000 out of 100,000 interrupts are spurious, the kernel loses patience, disables the interrupt line forevermore, and starts polling the line instead. There is a real cost to this action, which is why the kernel allows spurious interrupts to get to such a high proportion of the total. Once the response is triggered, there is no going back, even if the spurious interrupts were the result of a brief hardware glitch. 

With the adaptive polling mechanisms put into place to support the above features, the kernel is also able to take a more flexible approach to handling of spurious interrupts. 9,900 bad interrupts out of 10,000 are now enough to cause the spurious interrupt handling mechanism to kick in; as before, it disables the interrupt and begins polling. After a period, though, the new code will reenable the interrupt line, just to see what happens. If the source of spurious interrupts has stopped, the interrupt can be used as before. If, instead, spurious interrupts are still being delivered, the line will be blocked again for a longer period of time. 

There has not been a lot of discussion of this patch set so far; one [comment](<https://lwn.net/Articles/392143/>) worried that polling could cause users not to realize that there are problems in their systems. But Tejun says that this kind of response is required to get reasonably solid behavior out of flaky hardware, and nobody seems to want to challenge that claim. So it seems fairly likely that a future version of this patch will find its way into the mainline at some point.
