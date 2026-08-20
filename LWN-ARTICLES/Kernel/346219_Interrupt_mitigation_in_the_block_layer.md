---
title: Interrupt mitigation in the block layer
url: https://lwn.net/Articles/346219/
date: "August 10, 2009"
category: Block layer
author: "By Jonathan Corbet August 10, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 10, 2009

Network device drivers have been using the increasingly misnamed NAPI ("new API") interface for some time now. NAPI allows a network driver to turn off interrupts from an interface and go into a polling mode. Polling is often seen as a bad thing, but it's really only a problem when poll attempts turn up no useful work to do. With a busy network interface, there will always be new packets to process; "polling," in this situation, really means "going off to deal with the accumulated work." When there is always work to do, interrupts informing the system of that fact are really just added noise. Your editor likes to compare the situation to email notifications; anybody who gets a reasonable volume of email is quite likely to turn such notifications off. They are distracting, and there is probably always email waiting whenever one gets around to checking. 

NAPI is well suited to network drivers, since high packet rates can lead to high interrupt rates, but it has not spread to other parts of the kernel, where interrupt rates are lower. That situation could change in 2.6.32, though, if Jens Axboe follows through with his plan to merge the new [blk-iopoll](<http://lwn.net/Articles/346187/>) infrastructure into the mainline. In short, blk-iopoll is NAPI for block devices; indeed, some of the core code was borrowed from the NAPI implementation. 

Converting a block driver to the blk-iopoll is straightforward. Each interrupting device needs to have a `struct blk_iopoll` structure defined for it, presumably in the structure which describes the device within the driver. This structure should be initialized with: 

```
#include <linux/blk-iopoll.h>
    
        typedef int (blk_iopoll_fn)(struct blk_iopoll *, int);
    
        void blk_iopoll_init(struct blk_iopoll *iop, int weight, blk_iopoll_fn *poll_fn);
```

The `weight` value describes the relative importance of the device; a higher weight results in more requests being processed in each polling cycle. As with NAPI, there is no definitive guidance as to what `weight` should be; in Jens's initial patch, it is set to 32. The `poll_fn()` will be called when the block subsystem decides that it's time to poll for completed requests. 

I/O polling for a device is controlled with: 

```
void blk_iopoll_enable(struct blk_iopoll *iop);
        void blk_iopoll_disable(struct blk_iopoll *iop);
```

A call to `blk_iopoll_enable()` must be made by the driver before any polling of the device will happen. Enabling polling allows that polling to occur, but does not _cause_ it to happen. There is no point in polling a device which is not doing any work, so the block layer will not actually poll a given device until the driver informs it that there may be a reason to do so. 

That normally happens when the device is actually interrupting. The driver can, in its interrupt handler, switch over to polling mode through a three-step process. The first is to check the global variable `blk_iopoll_enabled`; if it is zero, block I/O polling cannot be used. Assuming polling is enabled, the driver should prepare the `blk_iopoll` structure with: 

```
int blk_iopoll_sched_prep(struct blk_iopoll *iop);
```

In the first version of the patch, a return value of zero means that the preparation "failed," either because polling is disabled or because the device is already in polling mode. In future versions, the sense of the return value is likely to be inverted to the more standard "zero means success" mode. If `blk_iopoll_sched_prep()` succeeds, the driver can then call: 

```
void blk_iopoll_sched(struct blk_iopoll *iop);
```

At this point, polling mode has been entered; the driver need only disable interrupts from its device and return. The "disable interrupts" step should, of course, be done at the device itself; masking the IRQ line would be an antisocial act in a world where those lines are shared. 

Later on, the block layer will call the `poll_fn()` which was provided to `blk_iopoll_init()`. The prototype for this function is: 

```
typedef int (blk_iopoll_fn)(struct blk_iopoll *iop, int budget);
```

The polling function is called (in software interrupt context) with `iop` being the related `blk_iopoll` structure, and `budget` being the maximum number of requests that the poll function should process. In normal usage, the driver's device-specific structure can be obtained from `iop` with `container_of()`. The `budget` value is just the `weight` that was specified back at initialization time. 

The return value should be the number of requests actually processed. If the device consumes less than the given `budget`, it should turn off further polling with: 

```
void blk_iopoll_complete(struct blk_iopoll *iopoll);
```

Interrupts from the device should be re-enabled, since further polling will not happen. Note that the block layer assumes that a driver will _not_ call `blk_iopoll_complete()` if it has consumed its full budget. If it's necessary to return to interrupt mode despite having exhausted the budget, the driver should either (1) use `blk_iopoll_disable()`, or (2) lie about the number of requests processed when returning from the polling function. 

One might well wonder about the motivation behind all of this work. Block device interrupt handling has not traditionally been a performance bottleneck. The problem is the rapid improvement in solid-state storage devices. It is expected that, before too long, these devices will be operating in the range of 100,000 I/O operations per second - far beyond anything that rotating storage can do. When dealing with that many I/O operations, the kernel must take care to minimize the per-operation overhead in any way possible. As others have observed, the block layer needs to become more like the network layer, with the per-request cost squeezed to a bare minimum. The blk-iopoll code is a step in that direction. 

How big a step? Jens has posted [some preliminary numbers](<https://lwn.net/Articles/346256/>) showing significant reductions in system time on a random-read disk benchmark. More testing will certainly be required; in particular, some developers are concerned about the possibility of increasing I/O latency. But the initial numbers suggest that this work has improved the efficiency of the block subsystem under load.
