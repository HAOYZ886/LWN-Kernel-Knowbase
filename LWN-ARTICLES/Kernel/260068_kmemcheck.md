---
title: kmemcheck
url: https://lwn.net/Articles/260068/
date: "November 27, 2007"
category: "Development tools-Kernel debugging"
author: "By Jonathan Corbet November 27, 2007"
---

By **Jonathan Corbet**  
November 27, 2007

Using uninitialized memory can lead to some seriously annoying bugs. If you are lucky, the kernel will crash with the telltale slab poisoning pattern (`0x5a5a5a5a` or similar) in the traceback. Other times, though, something more subtly wrong happens, forcing a long hunt for the stupid mistake. Wouldn't it be nicer if the kernel could simply detect references to uninitialized memory and scream loudly at the time? 

The [kmemcheck](<http://lwn.net/Articles/259974/>) patch recently posted by Vegard Nossum offers just that functionality, though, perhaps, in a somewhat heavy-handed manner. A kernel with kmemcheck enabled is unlikely to be suitable for production use, but it should, indeed, do a good job at finding code using memory which has not yet been set to a useful value. 

Kmemcheck is a relatively simple patch; the approach used is, essentially, this: 

  * Every memory allocation is trapped at the page-allocator level. For each allocation, the requested order is increased by one, doubling the size of the allocation. The additional ("shadow") pages are initialized to zero and kept hidden. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

  * The allocated memory is returned to the caller, but with the "present" bit cleared in the page tables. As a result, every attempt to access that memory will cause a page fault. 

  * Once the fault happens, kmemcheck (through some ugly, architecture-specific code) determines the exact address and size of the attempted access. If the access is a write, the corresponding bytes in the shadow page are set to `0xff` and the operation is allowed to complete. 

  * For read accesses, the corresponding shadow page bytes are tested; if any of them are zero, the code concludes that the read is trying to access uninitialized data. A stack traceback is printed to enable the developer to find the location where this access is happening. 

As should be evident, running with kmemcheck enabled will have certain performance impacts. Taking a page fault on every access to slab memory just cannot be fast. Doubling the size of every allocation will impose costs of its own, including the cache effects of simply working with twice as much memory. But that is a cost which can be paid when the kernel is being run in a debugging mode. 

Vegard has posted [some sample output](<http://folk.uio.no/vegardno/linux/kmemcheck-20071127.txt>) which shows how the system responds to reads from uninitialized memory. If this output is to be believed, access to unset memory is not an especially uncommon occurrence in current kernels. If some of references flagged here, once tracked down, turn out to be real bugs, the kmemcheck patch will have earned its keep, even if it never finds its way into the mainline.
