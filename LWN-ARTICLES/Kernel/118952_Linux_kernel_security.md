---
title: Linux kernel security
url: https://lwn.net/Articles/118952/
date: "January 12, 2005"
category: "Security-Vulnerabilities"
author: 
---

There has been a surprising series of kernel security problems reported over the last week. These include: 

  * The [uselib() vulnerability](<https://lwn.net/Articles/118955/>) disclosed by Paul Starzetz. A locking mistake in an old and mostly unused system call creates a race condition which can be exploited to change protections on memory - and compromise the system. The exploit has not been released, but Mr. Starzetz claims that the race is relatively easy to exploit by first consuming large amounts of memory to force the kernel to sleep in the right spot. 

  * Paul Starzetz also discovered [a race condition in the page fault handler](<https://lwn.net/Articles/118964/>) which can only be exploited on SMP systems. If two threads tried to expand the same downward-growing memory segment at the same time, the result could be an exploitable corruption of the page tables. 

  * The grsecurity team, frustrated at a seeming lack of interest in security problems among the kernel developers, [disclosed five vulnerabilities](<https://lwn.net/Articles/118251/>) at once. One of these is a denial-of-service problem where users could lock more than the authorized amount of memory into physical RAM; as it turns out, the kernel developers [still are not overly concerned](<https://lwn.net/Articles/118972/>) about that problem. The other vulnerabilities require root access (or at least access to physical devices) to exploit; one of them is in a driver which does not compile in 2.6. 

Fixes for the first two vulnerabilities have been merged into the pre-2.6.11 BitKeeper repository; the last set will be fixed as well, but with less urgency. Fixes can also be found in [the -ac tree](<https://lwn.net/Articles/118465/>) and in the updated kernels being issued by distributors. 

One concern that has been raised by these disclosures is that the new kernel development model, by encouraging such large changes between releases, is allowing the creation of more security problems. While that worry could yet prove to be justified, all of the vulnerabilities listed above, with the exception of the `RLIMIT_MEMLOCK` denial of service problem, are present in the 2.4 kernel as well. They were not introduced or enabled by the new development model. 

Another concern is more valid, however: the kernel development project does not have an official security contact or process for handling security problems. Developers who know how the kernel process works have no trouble getting consideration for security-related problems and patches, but the whole process looks far more opaque to the rest of the world. There is a clear need for an easily-found contact for kernel security issues. Chris Wright, who has done a fair amount of security-related kernel work, is [pushing for improvements in this area](<https://lwn.net/Articles/118976/>), and, most importantly, has volunteered to do much of the work. So chances are this problem will not last much longer.
