---
title: "ACPI, device interrupts, and suspend states"
url: https://lwn.net/Articles/146094/
date: "August 3, 2005"
category: ACPI; Device drivers; Power management
author: 
---

The 2.6.13-rc5 prepatch brought with it the reversal of a couple of ACPI-related patches. A look at what happened is rewarding in that it shows how hard it can be to get some things right, and how the kernel development model tries to address these issues. 

Earlier 2.6.13 prepatches included a change to the core ACPI system. Whenever the system (or a part of it) is being suspended, the modified ACPI code would break the link which routed device interrupts into the processor. This change is part of a new set of rules which expects every device to release its interrupt line on suspend, and to reacquire it on resume. There are a few reasons for wanting to do things this way: 

  * In theory, at least, a device could be resumed to find that its interrupt number has changed. People who reconfigure their hardware while the system is suspended (as opposed to being truly shut down) might be seen as actively looking for trouble, but it still might be nice to make things work for them when possible. 

  * The interrupt handler for a suspended device should not normally be called, but that can happen in the case of shared interrupts. Any interrupt handler which tries to access a suspended device is likely to run into problems; having every `suspend()` method release the device's interrupt line can help to avoid this situation. 

  * On resume, interrupts for a device whose driver has not yet been resumed may be seen as spurious and shut down. If that interrupt line is shared, however, other devices could be affected. This problem can be avoided by having ACPI shut down the interrupt altogether until individual drivers restore it, but that depends on drivers explicitly reallocating their interrupt lines. 

The problem with the ACPI change is that it breaks a large number of drivers, and, as a result, it breaks suspend on systems where it used to work. The power management hackers seem to see this situation as an unfortunate, but necessary step toward getting suspend working reliably on a much broader range of hardware. Having individual drivers release and reacquire their interrupts is also seen as necessary to support runtime power management - suspending of individual devices in a running system to save power. The ACPI change, it is said, fixes more systems than it breaks, and is thus worthwhile. 

Linus [disagreed](<https://lwn.net/Articles/146100/>) and reverted the patch, saying: 

The thing is, we're better off making very very slow progress that is _steady_, than having people who _used_ to have things work for them suddenly break. 

So I believe that if we fix two machines and break one machine, we've actually regressed. It doesn't matter that we fixed more than we broke: we _still_ regressed. Because it means that people can't trust the progress we make! 

The right solution, according to Linus, is to go ahead and add the `free_irq()` and `request_irq()` calls to individual drivers when it makes sense to do so, and when it does not break things for individual users. Meanwhile, however, the ACPI subsystem should still restore the interrupt state on resume so that unmodified drivers do not break. There are some remaining issues with how that is done: it may involve running the ACPI AML interpreter with interrupts disabled, which leads to a number of interesting situations. Benjamin Herrenschmidt also [pointed out](<https://lwn.net/Articles/146103/>) that it could lead to situations where drivers may not be able to receive interrupts during the resume process. 

Eventually, one assumes, these details will be worked out. In the mean time, it will be interesting to see if the "revert any patch that breaks somebody's machine" policy holds. If it leads to a more stable experience for Linux users, it seems like it would be a good thing.
