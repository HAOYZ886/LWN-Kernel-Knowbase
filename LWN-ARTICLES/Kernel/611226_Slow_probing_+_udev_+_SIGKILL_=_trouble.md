---
title: "Slow probing + udev + SIGKILL = trouble"
url: https://lwn.net/Articles/611226/
date: "September 9, 2014"
category: "Device drivers-Asynchronous probing; udev"
author: "By Jonathan Corbet September 9, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 9, 2014

A few years back, kernel developers briefly tried to make all device probing into an asynchronous activity. The resulting complications proved to be hard enough to resolve that the effort was deemed to not be worthwhile and the change was backed out. Now asynchronous probing is back on the table. The idea is not being received entirely warmly, but the problem it is trying to solve is real — and slightly strange. 

Things started in November 2013, when Tetsuo Handa [wrote a patch](<http://git.kernel.org/linus/786235eeba0e1e85e5cbbb9f97d1087ad03dfa21>) to make `kthread_create()`, a kernel function which creates and runs a kernel thread, killable (meaning that it will exit on a `SIGKILL` signal). Prior to this change, any process that was running in `kthread_create()` would be temporarily immune to `SIGKILL`; in particular, it would not respond if the out-of-memory (OOM) killer decided that it was in need of termination. While there is room for disagreement with the heuristics by which the OOM killer chooses victims, few developers believe that those victims, once chosen, should remain in the system for an arbitrary amount of time. With Tetsuo's change, a process that had caused the creation of a kernel thread would no longer be immune to the OOM killer's wrath. 

Interestingly, this patch caused some systems to fail to boot. A number of storage-system drivers require kernel threads to complete the process of probing for storage arrays. This probing process can involve a fair amount of work, to the point that it can take a minute or so to run. But it seems that systemd-udev does not have unlimited patience; it starts a 30-second timer (reduced from three minutes last year) when loading a device module, and kills the loading process (with `SIGKILL`) should that timer expire. So the process trying to probe the storage array is killed, the array assembly fails, and the system does not boot. Prior to Tetsuo's change, the signal would have been ignored during the probing process; afterward, it became fatal. Other types of drivers, such as those that must go through a lengthy firmware-downloading exercise, can also be affected by this problem. 

The resulting discussions were spread out across multiple lists and bug trackers and thus were somewhat hard to follow. Kernel developers seemed to be generally [of the opinion](<https://lwn.net/Articles/611312/>) that a hard-coded 30-second timeout in systemd-udev is not a good idea, and that the problem should be fixed there. The systemd developers [believe](<http://article.gmane.org/gmane.comp.sysutils.systemd.devel/17860>) that any module taking more than 30 seconds to load is simply buggy and should be fixed. Tetsuo [suggested](<https://launchpadlibrarian.net/169657493/kthread-defer-leaving.patch>) that `kthread_create()` could delay its exit for ten seconds on `SIGKILL` if that signal originates anywhere other than the OOM killer. None of these ideas have found a consensus or led to a solution to the problem. 

Of course, there is the option of simply increasing the timeout in the configuration files, something that was [done by the device mapper developers](<http://www.redhat.com/archives/lvm-devel/2013-September/msg00036.html>) in response to a similar problem. But that approach strikes nobody as being particularly elegant. 

There is one other way around this difficulty: device drivers could, at module load time, simply register themselves and do only the work that can be completed quickly. Any time-consuming work would be pushed off into a separate thread to run asynchronously, after module loading is done and systemd-udev has left the vicinity. This type of asynchronous initialization might also have the effect of improving boot times by allowing other work to happen while the slow probing work is being done. 

To this end, Luis Rodriguez has posted [a patch set](<https://lwn.net/Articles/610878/>) adding asynchronous probing to the driver core. The patches add a new field (`async_probe`) to `struct device_driver`; if that field has a true value, probing for devices will happen asynchronously by way of a workqueue. Three drivers (pata_marvell, cxgb4, and mptsas) were modified to request the new asynchronous behavior. There is also a variant of Tetsuo's 10-second-delay patch that is primarily intended to put a warning into the system log should a non-OOM-killer `SIGKILL` show up in `kthread_create()`; it is there to help identify other drivers that need to be converted to the asynchronous mode. 

Tejun Heo, who, among other things, maintains the serial ATA layer, was not fond of this approach. His opposition, in the end, [comes down](<https://lwn.net/Articles/611235/>) to two issues: 

  * Any driver can potentially exhibit this problem. Taking a whack-a-mole approach by tweaking drivers with reported issues is thus the wrong way to solve the problem — there will always be more drivers that still need fixing. 

  * Linux systems have always managed to have locally attached storage devices available when module loading completes. Moving to an asynchronous mode is thus a user-visible behavior change; it could well break administrative scripts that expect storage arrays to be ready for mounting immediately after the driver has been loaded. 

The second issue is the more controversial of the two. Modern distributions tend not to make assumptions about when devices will show up in the system, so some developers [argue](<https://lwn.net/Articles/611313/>) that there should no longer be any problems in this area. But old administrative scripts can hang around for a long time. So the risk of breaking real-world systems with this kind of change is real, even if it is not clear how many systems might be affected. 

Of course, other real-world systems are broken now, so something needs to be done. The most likely outcome would appear to be some sort of asynchronous probing that is done under user-space control; unless user space has explicitly requested it, the behavior would not change. The [probable implementation](<https://lwn.net/Articles/611311/>) of this approach is a global flag that turns on asynchronous device probing, with one exception. There is a good chance that any kernel code that calls `request_module()` expects that the requested module's devices will be available when the call returns. So modules loaded via this path will, for now at least, have to be loaded synchronously. 

On distributions where the management of storage arrays is done with distribution-supplied scripts, the "use asynchronous probing" switch could be turned on by default. Others would require some sort of manual intervention. It is not the best resolution that one might imagine, but it might be the best that is on offer in the near future.
