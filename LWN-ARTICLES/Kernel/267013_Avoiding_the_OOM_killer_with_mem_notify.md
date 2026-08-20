---
title: Avoiding the OOM killer with mem_notify
url: https://lwn.net/Articles/267013/
date: "January 30, 2008"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jake Edge January 30, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
January 30, 2008

Having applications that use up all the available memory can be a fairly painful experience. For Linux systems, it generally means a visit from the out-of-memory (OOM) killer, which will try to find processes to kill. As one would guess, coming up with rules governing which process to kill is challenging--someone, somewhere, will always be unhappy with a choice the OOM killer makes. Avoiding it altogether is the goal of the [mem_notify](<http://lwn.net/Articles/266586/>) patch. 

When memory gets tight, it is quite possible that applications have memory allocated--often caches for better performance--that they could free. After all, it is generally better to lose some performance than to face the consequences of being chosen by the OOM killer. But, currently, there is no way for a process to know that the kernel is feeling memory pressure. The patch provides a way for interested programs to monitor the `/dev/mem_notify` file to be notified if memory starts to run low. 

`/dev/mem_notify` is a character device that signals memory pressure by becoming readable. Interested programs can open the file and then use `poll()` or `select()` to monitor the file descriptor. Alternatively, signal-driven I/O can be enabled via the `FASYNC` flag and the system will deliver a SIGIO signal to the process when the device becomes readable. If it becomes readable, the process should free any memory that it can afford to give up. If enough memory is freed this way, the kernel will have no need to call in the OOM killer. 

The crux of the patch is how to decide that memory pressure is occurring. mem_notify modifies `shrink_active_list()` to look for movement of an anonymous page to the inactive list, which is an indication that some will likely be swapped out soon. When that occurs, `memory_pressure_notify()` (with the pressure flag set to 1) will be called for that zone. When the number of free pages for the zone increase above a threshold--based on `pages_high` and `lowmem_reserve` for the zone--`memory_pressure_notify()` is called again, but with the pressure flag set to 0, effectively ending the memory pressure event for that zone. 

If there are numerous processes waiting for a memory pressure notification, it could be counterproductive to wake them all at once--the "thundering herd" problem. To combat this, the patch set adds the ability to wake fewer processes than are waiting on the poll event, by adding the `poll_wait_exclusive()` function. `poll_wait_exclusive()` will in turn call `add_wait_queue_exclusive()` so that a member of the `wake_up()` family can be used that will limit the number of processes woken up. Previously, only `poll_wait()` was available, it uses `add_wait_queue()`, which does not provide this ability. Also, to reduce the frequency of processes waking up to reclaim memory, `memory_pressure_notify()` will only do that once every five seconds. 

The `/proc/zoneinfo` output has been changed to include the mem_notify status. This can be used by a human for diagnostic purposes or by a program to check the current status of zones for memory pressure. 

The embedded community has a lot of interest in seeing this feature get added to the kernel. Devices like phones and PDAs are often running close to their memory limits and the OOM killer is currently unavoidable when the user opens yet another application. With this patch in place, programs that use a lot of memory, but could get by with less, can be changed to free up their caches and the like when memory gets tight. As memory hungry programs get changed, other users will benefit as well. 

The patch, submitted by Kosaki Motohiro, has been through several iterations on linux-kernel. The work was originally started by Marcelo Tosatti, with the fifth version recently posted by Kosaki. Previous versions have been well received and with relatively few comments on this iteration, it would seem to be getting close to being merged.
