---
title: The mempressure control group proposal
url: https://lwn.net/Articles/531077/
date: "January 3, 2013"
category: "Control groups; Memory management-Control groups"
author: "January 3, 2013 This article was contributed by Bartlomiej Zolnierkiewicz"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

January 3, 2013

This article was contributed by Bartlomiej Zolnierkiewicz

Last November, LWN described [the vmpressure_fd() work](<https://lwn.net/Articles/524742/>) which implemented a new system call making it possible for user-space applications to be informed when system memory is tight. Those applications could then respond by freeing memory, easing the crunch. That patch set has since evolved considerably. 

Based on the feedback that author Anton Vorontsov received, the concept has changed from a new system call to [a new, control-group-based subsystem](<https://lwn.net/Articles/528687/>). The controller implementation allows for integration with the memory controller, meaning that applications can receive notifications when their specific control group is running low on memory, even if the system as a whole is not under memory pressure. 

As with previous versions of the patch, applications can receive notifications for three levels of memory pressure: "low" (memory reclaim is happening at a low level), "medium" (some swapping is happening), and "oom" (memory pressure is severe). But these notifications may no longer be the primary way in which applications interact with the controller, thanks to the most significant change in comparison to the previous `vmpressure_fd()` solution: the addition of a user-space "shrinker" interface allowing the kernel to ask user space to free specific amounts of memory when needed. This API was inspired by Andrew Morton's [feedback](<https://lwn.net/Articles/531138/>) on the first revision of the mempressure control group subsystem patch: 

It blurs the question of "who is in control". We tell userspace "hey, we're getting a bit tight here, please do something". And userspace makes the decision about what "something" is. So userspace is in control of part of the reclaim function and the kernel is in control of another part. Strange interactions are likely. 

Andrew also worried that application developers may tune their programs against a particular kernel version; subtle behavioral changes in new kernel releases might then cause regressions. In short, Andrew complained, the behavior of the system as a whole was not testable, so there would be no way to know if subsequent kernel changes made performance worse. 

Andrew's suggestion was to give more control to the kernel and introduce some kind of interface for user-space memory scanning and freeing (similar in its main concept to the `shrink_slab()` kernel shrinkers). This interface would control user-space reclaim behavior; if something goes wrong, it will be up to kernel to resolve the issue. It would also give kernel developers the ability to test and tune whole system behavior by writing a compliant user-space test application and running it. 

The user-space shrinker implementation by Anton operates on the concept of chunks of an application-defined size. There is an assumption that the application does memory allocations with a specific granularity (the "chunk size," which may be not 100% accurate but the more accurate it is, the better). So if the application caches data in chunks of 1MB, that is the size it will provide to the shrinker interface. That is done through a sequence like this: 

  1. The application opens the control interface, which is found as the file `cgroup.event_control` in the controller directory. 

  2. The shrinker interface (`mempressure.shrinker` in the controller directory) must also be opened. 

  3. The `eventfd()` system call is used to obtain a third file descriptor for notifications. 

  4. The application then writes a string containing the `eventfd()` file descriptor number, the `mempressure.shrinker` file descriptor number, and the chunk size to the control interface. 

Occasionally, the application should write a string to the shrinker file indicating how many chunks have been allocated or (using a negative count) freed. The kernel uses this information to maintain an internal count of how many reclaimable chunks the application is currently holding on to. 

If the kernel wants the application to free some memory, the notification will come through the `eventfd()` file descriptor in the form of an integer count of the number of chunks that should be freed. The kernel assumes that the application will free the specified number of chunks before reading from the `eventfd()` file descriptor again. If the application isn't able to reclaim all chunks for some reason, it should re-add the number of chunks that were not freed by writing to the `mempressure.shrinker` file. 

The patchset also includes [an example application](<https://lwn.net/Articles/531140/>) (slightly buggy in the current version) for testing the new interface. It creates two threads; the first thread initializes the user-space shrinker mechanism notifications and then tries to allocate memory (more than physically available) in an infinite loop. The second thread listens for user-space shrinker notifications and frees the requested number of chunks (also in an infinite loop). Ideally, during the run of the test application the system shouldn't get into an out-of-memory condition and it also shouldn't use much swap space (if any is available of course). 

Various comments were received on the patch set, so at least one more round of changes will be required before this interface can be considered for merging into the mainline. There is also an open question on how this feature interacts with [volatile ranges](<https://lwn.net/Articles/522135/>) and whether both mechanisms (neither of which has yet been merged) are truly required. So this discussion may continue well into the new year before we end up with reclaimable user-space memory caches in their final form.
