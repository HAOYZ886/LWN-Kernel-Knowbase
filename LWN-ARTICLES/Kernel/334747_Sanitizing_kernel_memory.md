---
title: Sanitizing kernel memory
url: https://lwn.net/Articles/334747/
date: "May 27, 2009"
category: "Security-Kernel hardening"
author: "By Jake Edge May 27, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 27, 2009

The contents of memory consist of vast quantities of useless--to an attacker at least--data, along with a small amount that would be of interest. Cryptographic keys, passwords, and the like are probable targets of those with malicious intent. Normally, the kernel guards memory from access by unprivileged processes, but, various kernel bugs have sometimes allowed memory information to leak. A recently proposed [patch](<https://lwn.net/Articles/334526/>) would eliminate a specific subset of those kinds of leaks by "sanitizing" pages as they are freed. 

Larry Highsmith adapted code from the [PaX project](<http://pax.grsecurity.net/>) to add a flag to kernel pages marking them as "sensitive" pages. The pages would then be cleared as they were freed, so that any information leak from those pages would be useless. As part of the justification for the change, Highsmith noted Stanford University [paper entitled "Shredding Your Garbage: Reducing Data Lifetime Through Secure Deallocation"](<http://www.stanford.edu/~blp/papers/shredding.html/>) as well as the ["cold boot"](<http://citp.princeton.edu/memory/>) attacks to recover memory from powered-down systems. 

Highsmith's patch would eliminate cases where freed memory contents leak, either via a kernel bug or some other means, by clearing the page as it is freed, but only for memory marked as sensitive. The four additional patches in his original series then applied the sensitive flag to various kernel subsystems (crypto, audit, and key handling). 

While the kernel hackers were generally agreeable to the idea of sanitizing memory, there were a number of objections to Highsmith's first attempt. A trivial one, which was fixed in later patches, was a Signed-off-by line that didn't give his full name (just "Larry H."). As the PaX project is developed by the pseudonymous "PaX Team"--thus not able to fulfill the requirements for a kernel sign off--several folks were quick to point out that a full name was required. More substantive objections were heard about using up a scarce resource in the form of a page flag. Alan Cox [pointed out](<https://lwn.net/Articles/334904/>) that a virtual memory area (VMA) flag would work as well, or that places in the patch that set the flag could just clear the memory instead: 

[...] page flags are very precious, very few and if we run out will cost us a vast amount of extra kernel memory. If page flags were free the question would be trivial - but they are not. Thus it is worth asking whether its actually harder to remember to zap the buffer or set and clear the flag correctly. 

There was a bit of a digression into the security issues surrounding suspend and hibernate, with Highsmith claiming that security conscious users just disabled that functionality altogether. Cox and Pavel Machek disagreed, noting the ability to encrypt the images that get written to disk with today's hibernate code. Cox was also [concerned](<https://lwn.net/Articles/334913/>) that marking things as sensitive makes an attacker's job easier: 

If you've got a rogue module you already lost, except that by marking what is sensitive you made the bad guys job easier. Bit like the way people visually overlay maps and overhead shots from multiple sources and the 'scrubbed' secret locations stand out for you and are easier to find than if they were left. 

In the end, any memory the kernel handles is potentially sensitive. Some applications--notably GPG--take great pains to try to ensure that their memory is not swapped and is cleared of keys and other sensitive data when they are no longer needed. As Ingo Molnar [put it](<https://lwn.net/Articles/334914/>): ""The whole kernel contains data that 'should not be leaked'."" This led to a new approach: for users who want sanitized pages--based on the `sanitize_mem` boot time parameter--simply clear _all_ pages when they are freed. A [much smaller patch](<https://lwn.net/Articles/334919/>) that implemented that scheme was then posted by Highsmith. 

In addition, there are kernel allocations that are for objects smaller than a page which could contain sensitive data. Highsmith has also [submitted](<https://lwn.net/Articles/334921/>) changes to `kfree()` and `kmem_cache_free()` that would clear these objects as they are freed. In the end, with both of these patches applied in a kernel with `sanitize_mem` enabled, all _free_ kernel memory will be cleared. But, of course, as several folks pointed out, in many cases the memory of interest will still be in use. 

Certainly a kernel with sanitized memory is more resistant to leaking memory contents, but depending on the threat one is trying to defend against, it may not be enough. The physical attacks against memory contents (i.e. "cold boot") are still likely to be effective--though free memory won't be recoverable--and other kinds of bugs could still leak memory in use. Highsmith presented an [analysis](<https://lwn.net/Articles/334924/>) of kernel information leaks, which was partially based on this interesting [list of CVEs and git commits](<http://web.mit.edu/tabbott/www/cve-data/cve-data.txt>) that fixed them. In it, there were a half-dozen examples of information leaks that would have been prevented by his changes. 

No further objections have been noted, and the patches are not terribly intrusive, so it would seem there is some chance they might make their way into 2.6.31.
