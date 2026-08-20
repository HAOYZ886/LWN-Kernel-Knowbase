---
title: A new suspend/hibernate infrastructure
url: https://lwn.net/Articles/274008/
date: "March 19, 2008"
category: Device drivers; Power management; Software suspend
author: "By Jonathan Corbet March 19, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 19, 2008

While attending conferences, your editor has, for some years, made a point of seeing just how many other attendees have some sort of suspend and resume functionality working on their laptops. There is, after all, obvious value in being able to sit down in a lecture hall, open the lid, and immediately start heckling the speaker via IRC without having to wait for the entire bootstrap sequence to unfold. But, regardless of whether one is talking about suspend-to-RAM ("suspend") or suspend-to-disk ("hibernation"), there are surprisingly few people using this capability. Despite the efforts which have been made by developers and distributors, suspend and hibernate still just do not work reliably for a lot of people. 

For your editor, suspend _always_ works, but the success rate of the resume operation is about 95% - just enough to keep using it while inspiring a fair amount of profanity in inopportune places. 

Various approaches to fixing suspend and hibernation have been proposed; these include [TuxOnIce](<http://tuxonice.org/>) and [kexec jump](<http://lwn.net/Articles/242107/>). Another possibility, though, is to simply fix the code which is in the kernel now. There is a lot that has to be done to make that goal a reality, including making the whole process more robust and separating the suspend and hibernation cases which, as Linus has stated rather strongly several times, are really two different problems. To that end, Rafael Wysocki has posted [a new suspend and hibernation infrastructure for devices](<http://lwn.net/Articles/273660/>) which has the potential to improve the situation - but at a cost of creating no less than 20 separate device callbacks. 

For the (relatively) simple suspend case, there are four basic callbacks which should be provided in the new `pm_ops` structure by each bus and, eventually, by every device: 

```
int (*prepare)(struct device *dev);
        int (*suspend)(struct device *dev);
    
        int (*resume)(struct device *dev);
        void (*complete)(struct device *dev);
```

When the system is suspending, each device will first see a call to its `prepare()` callback. This call can be seen as a sort of warning that the suspend is coming, and that any necessary preparation work should be done. This work includes preventing the addition of any new child devices and anything which might require the involvement of user space. Any significant memory allocations should also be done at this time; the system is still functional at this point and, if necessary, I/O can be performed to make memory available. What should _not_ happen in `prepare()` is actually putting the device into a low-power state; it needs to remain functional and available. 

As usual, a return value of zero indicates that the preparation was successful, while a negative error code indicates failure. In cases where the failure is temporary (a race with the addition of a new child device is one possibility), the callback should return `-EAGAIN`, which will cause a repeat attempt later in the process. 

At a later point, `suspend()` will be called to actually power down the device. With the current patch, each device will see a `prepare()` call quickly followed by `suspend()`. Future versions are likely to change things so that all devices get a `prepare()` call before any of them are suspended; that way, even the last `prepare()` callback can count on the availability of a fully-functioning system. 

The resume process calls `resume()` to wake the device up, restore it to its previous state, and generally make it ready to operate. Once the resume process is done, `complete()` is called to clean up anything left over from `prepare()`. A call to `complete()` could also be made directly after `prepare()` (without an intervening suspend) if the suspend process fails somewhere else in the system. 

The hibernation process is more complicated, in that there are more intermediate states. In this case, too, the process begins with a call to `prepare()`. Then calls are made to: 

```
int (*freeze)(struct device *dev);
        int (*poweroff)(struct device *dev);
```

The `freeze()` callback happens before the hibernation image (the system image which is written to persistent store) is created; it should put the device into a quiescent state but leave it operational. Then, after the hibernation image has been saved and another call to `prepare()` made, `poweroff()` is called to shut things down. 

When the system is powered back up, the process is reversed through calls to: 

```
int (*quiesce)(struct device *dev);
        int (*restore)(struct device *dev);
```

The call to `quiesce()` will happen early in the resume process, after the hibernation image has been loaded from disk, but before it has been used to recreate the pre-hibernation system's memory. This callback should quiet the device so that memory can be reassembled without being corrupted by device operations. A call to `complete()` will follow, then a call to `restore()`, which should put the device back into a fully-functional state. A final `complete()` call finishes the process. 

There are still two more hibernation-related callbacks: 

```
int (*thaw)(struct device *dev);
        int (*recover)(struct device *dev);
```

These functions will be called when things go wrong; once again, each of these calls will be followed by a call to `complete()`. The purpose of `thaw()` is to undo the work done by `freeze()` or `quiesce()`; it should put the device back into a working state. The `recover()` call will be made if the creation of the hibernation image fails, or if restoring from that image fails; its job is to clean up and get the hardware back into an operating state. 

For added fun, there are actually two sets of `pm_ops` callbacks. One is for normal system operation, but there is another set intended to be called when interrupts are disabled and only one CPU is operational - just before the system goes down or just after it comes back up. Clearly, interactions with devices will be different in such an environment, so different callbacks make sense. But the result is that fully 20 callbacks must be provided for full suspend and hibernate functionality. These callbacks have been added to the `bus_type` structure as: 

```
struct pm_ops *pm;
        struct pm_ops *pm_noirq;
```

Fields by the same name have also been added to the `pci_driver` structure, allowing each device driver to add its own version of these callbacks. For now, the old PCI driver `suspend()` and `resume()` callbacks will be used if the `pm_ops` structures have not been provided, and no drivers have been converted (at least in the patch as posted). 

As of this writing, discussion of the patch is hampered by an outage at vger.kernel.org. There are some concerns, though, and things are likely to change in future revisions. Among other things, the number of "no IRQ" callbacks may be reduced. But, with luck, the final resolution will leave us all in a position where suspend and hibernate work reliably.
