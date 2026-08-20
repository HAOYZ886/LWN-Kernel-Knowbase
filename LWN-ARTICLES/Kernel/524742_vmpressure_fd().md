---
title: vmpressure_fd()
url: https://lwn.net/Articles/524742/
date: "November 14, 2012"
category: Memory management
author: "By Jonathan Corbet November 14, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 14, 2012

One of the nice features of virtual memory is that applications do not have to concern themselves with how much memory is actually available in the system. One need not try to get too much work done to realize that some applications (or their developers) have taken that notion truly to heart. But it has often been suggested that the system as a whole would work better if interested applications could be informed when memory is tight. Those applications could react to that news by reducing their memory requirements, hopefully heading off thrashing or out-of-memory situations. The latest proposal along those lines is a new system call named `vmpressure_fd()`; it is unlikely to be merged in its current form, but it still merits a look. 

The idea behind Anton Vorontsov's [`vmpressure_fd()` patch set](<https://lwn.net/Articles/524299/>) is to create a mechanism by which the kernel can inform user space when the system is under memory pressure. An application using this call would start by filling out a `vmpressure_config` structure: 

```
#include <linux/vmpressure.h>
    
        struct vmpressure_config {
        	__u32 size;
    	__u32 threshold;
        };
```

The `size` field should hold the size of the structure; it is there as a sort of versioning mechanism should more fields be added to the structure in the future. The `threshold` field indicates the minimum level of notification the application is interested in; the available levels are: 

> `VMPRESSURE_LOW`
>      The system is out of free memory and is having to reclaim pages to satisfy new allocations. There is no particular trouble in performing that reclamation, though, so the memory pressure, while non-zero, is low. 
> 
> `VMPRESSURE_MEDIUM`
>      A medium level of memory pressure is being experienced — enough, perhaps, to cause some swapping to occur. 
> 
> `VMPRESSURE_OOM`
>      Memory pressure is at desperate levels, and the system may be about to fall prey to the depredations of the out-of-memory killer. 

An application might choose to do nothing at low levels of memory pressure, clean up some low-value caches at the medium level, and clean up everything possible at the highest level of pressure. In this case, it would probably set `threshold` to `VMPRESSURE_MEDIUM`, since notifications at the `VMPRESSURE_LOW` level are not actionable. 

Signing up for notifications is a simple matter: 

```
int vmpressure_fd(struct vmpressure_config *config);
```

The return value is a file descriptor that can be read to obtain pressure events in this format: 

```
struct vmpressure_event {
            __u32 pressure;
        };
```

The current interface only supports blocking reads, so a `read()` on the returned file descriptor will not return until a pressure notification has been generated. Applications can use `poll()` to determine whether a notification is available; the current patch does not support asynchronous notification via the SIGIO signal. 

Internally, the virtual memory subsystem has no simple concept of memory pressure, so the patch has to add one. To that end, the "reclaimer inefficiency index" is calculated by looking at the number of pages examined by the reclaimer and how many of those pages could _not_ be reclaimed. The need to look at large numbers of pages to find reclaim candidates indicates that reclaimable pages are getting hard to find — that the system is under memory pressure in other words. The index is simply the ratio of reclamation failures to the number of pages examined, expressed as a percentage. 

This percentage is calculated over a "window" of pages examined; by default, it is generated each time the reclaimer looks at 256 pages. This value can be changed by tweaking the new `vmevent_window` sysctl knob. There are also controls to set the levels at which the various notifications occur: `vmevent_level_medium` (default 60) and `vmevent_level_oom` (default 99); the "low" level is wired at zero, so it will trigger anytime that the system is actively looking for pages to reclaim. 

An additional mechanism exists to detect the out-of-memory case, since it can be hard to distinguish it using just the reclaimer inefficiency index. The reclaim code includes the concept of a "priority" which controls how aggressive it can be to reclaim pages; its value starts at 12 and falls over time as attempts to locate enough pages fail. If the priority falls to four (by default; it can be set with the `vmevent_level_oom_priority` knob), the system is deemed to be heading into an out-of-memory state and the notification is sent. 

Some reviewers questioned the need for a new system call. We already have a system call — `eventfd()` — that exists to create file descriptors for notifications from the kernel. Actually using `eventfd()` tends to involve an interesting dance where the application gets a file descriptor from `eventfd()`, opens a sysfs file, and writes the file descriptor number into the sysfs file to connect it to a specific source of events. But it is an established pattern that might be best maintained here. Another reviewer [suggested](<https://lwn.net/Articles/524745/>) using the perf events subsystem, but Anton [believes](<https://lwn.net/Articles/524747/>), not without reason, that perf brings a lot of complexity to something that should be relatively simple. 

The other complaint has to do with the integration (or lack thereof) with the "memcg" control-group-based memory usage controller. Memcg already has a notification mechanism (described in [Documentation/cgroups/memory.txt](<https://lwn.net/Articles/516022/>)) that can inform a process when a control group is running out of memory; it might make sense to use the same mechanism for this purpose. Anton [responded](<https://lwn.net/Articles/524748/>) that the memcg mechanism does not provide the same information, it does not account for all memory use, and that it requires the use of control groups — not always a popular kernel feature. Still, even if `vmpressure_fd()` is merged as a separate mechanism, it will at least have to be extended to work at the control group level as well. The code shows that this integration has been thought about, but it has not yet been implemented. 

Given these concerns, it seems unlikely that the current patch set will find its way into the mainline. But there is a clear desire for this kind of functionality in all kinds of use cases from very large systems to very small ones (Anton's patches were posted from a linaro.org address). So, one way or another, a kernel in the near future will probably have the ability to inform processes that it is experiencing some memory pressure. The next challenge will then be getting applications to use those notifications and reduce that pressure.
