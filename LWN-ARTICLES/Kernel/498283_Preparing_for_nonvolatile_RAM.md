---
title: Preparing for nonvolatile RAM
url: https://lwn.net/Articles/498283/
date: "May 23, 2012"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet May 23, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 23, 2012

Once upon a time, your editor had a job that involved working with a Data General Nova system. The Nova had an interesting characteristic: since it contained true core memory, the contents of that memory would persist across a reboot—or a power-down. So the end-of-day shutdown procedure was a simple matter of turning the machine off; when it was turned on the next morning, it would simply continue where it was before. There were no complaints about system boot time with that machine. The replacement of core memory with silicon-based RAM brought a lot of nice advantages, but the nonvolatile nature was lost on the way. But it appears that nonvolatile memory may be about to make a comeback, bringing some interesting development problems with it. 

Matthew Wilcox [raised the issue](<https://lwn.net/Articles/498289/>), noting that nonvolatile memory (NVM) is coming, that it promises bandwidth and latency numbers similar to those offered by dynamic RAM, and that, being cheaper than DRAM, it is likely to be offered in larger sizes than DRAM is. He later [disclaimed any resemblance](<https://lwn.net/Articles/498291/>) between this description and any future products to be offered by his employer; it is, he says, simply where the industry is going. Given that, it would be a good idea for the kernel community to be ready for this technology when it arrives. 

One part of being ready is figuring out how to deal with nonvolatile memory within the kernel. The suggested approach was to use a filesystem: 

We think the appropriate way to present directly addressable NVM to in-kernel users is through a filesystem. Different technologies may want to use different filesystems, or maybe some forms of directly addressable NVM will want to use the same filesystem as each other. 

A filesystem approach would allow the association of names with regions of NVM space; an API was then proposed to allow the kernel to perform tasks like mapping regions of NVM into the kernel's address space. 

One question that came up quickly was: won't the use of the filesystem model slow things down? There is a lot of overhead in the block layer, which was not designed to deal with "storage" that operates at full memory bandwidth. Matthew was never thinking of bringing in the full block layer, though; instead, he [said](<https://lwn.net/Articles/498295/>): ""I'm hoping that filesystem developers will indicate enthusiasm for moving to new APIs"". Such enthusiasm was in short supply in this discussion; that is probably more indicative of a lack of thought about the problem than any sort of active opposition (which was also in short supply). 

James Bottomley, though, [questioned the filesystem idea](<https://lwn.net/Articles/498297/>), suggesting that NVM should be treated like memory. He said that the way to access NVM might be through the kernel's normal memory APIs, with nonvolatility just being another attribute of interest. One could imagine calling `kmalloc()` with a new `GFP_NONVOLATILE` flag, for example. The only problem with this approach is that it is not enough to request an arbitrary nonvolatile region; callers will usually want a _specific_ NVM region that, presumably, contains data from a previous use. So the memory API would have to be extended with some sort of namespace giving reliable access to persistent data. To many, that namespace looks like a filesystem; James suggested using 32-bit keys like the SYSV shared memory mechanism does, but admirers of SYSV IPC tend to be scarce indeed on linux-kernel. 

So, while there are a lot of details to be worked out, some sort of name-based kernel API seems certain to come about. Then there will be a mechanism, either through the memory-related or filesystem-related system calls, to make NVM available to user space. But that leads to another, perhaps harder question: what, then, do we do with all that fast, nonvolatile memory? 

Some of it, certainly, could be used for caching; technologies like [bcache](<https://lwn.net/Articles/497024/>) could make good use of it. The page cache could go there; Matthew suggested that the inode cache might be another possibility. Both could speed booting considerably, though it would be necessary to somehow validate the cache contents against filesystems that could have changed while the system was down. Boaz Harrosh [suggested](<https://lwn.net/Articles/498299/>) that filesystems could store their journals in that space, speeding journal access and reducing journal I/O load on the underlying storage devices. He also mentioned checkpointing the system to NVM, allowing for quick recovery should the system go down unexpectedly. Vyacheslav Dubeyko had [some wilder ideas](<https://lwn.net/Articles/498300/>) about how NVM could eliminate system bootstrap entirely and make the concept of filesystems obsolete; instead, everything would just live in a persistent object environment. 

Clearly, many of these ideas are going to take some time to come to fruition. Nonvolatile memory changes things in fundamental ways; Linux may have to scramble to keep up, but, then, that is a high-quality problem to have. It will be most interesting to watch how this plays out over the coming years.
