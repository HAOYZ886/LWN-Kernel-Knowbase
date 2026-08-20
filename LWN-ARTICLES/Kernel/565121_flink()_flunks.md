---
title: flink() flunks
url: https://lwn.net/Articles/565121/
date: "August 28, 2013"
category: flink
author: "By Jonathan Corbet August 28, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 28, 2013

A few weeks back, we [noted](<https://lwn.net/Articles/562488/>) the merging of a patch adding support for the `flink()` system call, which would allow a program to link to a file represented by an open file descriptor. This functionality had long been blocked as the result of security worries; it went through this time on the argument that the kernel already provided that feature via the `linkat()` call. At that point, it seemed like this feature was set for the 3.11 kernel. 

Not everybody was convinced, though; in particular, Brad Spengler apparently raised the issue of processes making links to file descriptors that had been passed in from a more privileged domain. So Andy Lutomirski, the author of the original patch, posted [a followup](<https://lwn.net/Articles/565122/>) to restrict the functionality to files created with the `O_TMPFILE` option. The only problem was that nobody much liked the patch; Linus was [reasonably clear](<https://lwn.net/Articles/565123/>) about his feelings. 

What followed was a long discussion on how to better solve the problem, with a number of patches going back and forth. About the only thing that became clear was that the best solution was not yet clear. So, on August 28, Linus [reverted the original patch](<https://lwn.net/Articles/565125/>), saying ""Clearly we're not done with this discussion, and the patches flying around aren't appropriate for 3.11"" So the `flink()` functionality will have to wait until at least 3.12.
