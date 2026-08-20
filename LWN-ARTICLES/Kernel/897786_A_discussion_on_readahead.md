---
title: A discussion on readahead
url: https://lwn.net/Articles/897786/
date: "June 15, 2022"
category: Readahead
author: "By Jake Edge June 15, 2022 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 15, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

[Readahead](<https://lwn.net/Articles/888715/>) is an I/O optimization that causes the system to read more data than has been requested by an application—in the belief that the extra data will be requested soon thereafter. At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Matthew Wilcox led a session to discuss readahead, especially as it relates to network filesystems, with assistance from Steve French and David Howells. The latency of the underlying storage needs to factor into the calculation of how much data to read in advance, but it is not entirely clear how to do so. 

Wilcox began by describing readahead a bit. If user space is reading a file one byte at a time, Linux does not actually read the data that way; instead, it issues reads for a bigger chunk, say 64KB, which gets stored in the page cache. There is a certain amount of latency between the time a page is requested from the storage and when it appears in the page cache; that latency varies greatly over the wide variety of storage types that Linux supports. For network storage, those types can range from locally stored data on multi-gigabit Ethernet to data stored halfway around the world over decidedly slower links. Similarly, for local storage it can range from a 5GB-per-second NVMe SSD to some "crappy USB key picked up from a vendor at a trade show". There is "a lot of stuff to contend with there". 

[ ![\[Matthew Wilcox\]](https://static.lwn.net/images/2022/lsfmm-wilcox-sm.png) ](<https://lwn.net/Articles/897916/>)

In his experience, block-layer developers tend to do all of their testing using direct I/O; they think that "the page cache sucks" so they avoid it in their testing. James Bottomley said they are often trying to exclude the effects of the page cache from their testing in order to eliminate variables outside of their control. Wilcox said that it was unfortunate, since the performance including the page cache is "what the users are actually seeing"; it would be nice to notice problems where either too much or too little readahead is affecting the performance—before users do. 

He said that he has a [KernelNewbies wiki page](<https://kernelnewbies.org/MatthewWilcox/ImprovedReadaheadAlgorithm>) where he is collecting his thoughts about readahead and the page cache in general. The page cache "is awesome in some ways, in other ways it's terrible", but it would be good to fix the problems that it has. The Android developers encountered a problem with readahead, but "they worked around it in the worst way possible". They changed a setting for readahead, moving it from 256KB to several hundred megabytes, he said, in order to shave "some fraction of a second" from application startup time. That has other effects, of course; when they tried to upstream the patches, the memory-management developers said "no". 

Howells suggested that the Android developers should be using [`fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>) (or, he amended, [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>)) to indicate that the file should have more aggressive readahead. But Wilcox did not agree: "can we stop trying to pretend that user space knows what it is doing?" Android is a specialized environment, Bottomley said; it uses a log-structured filesystem, so the workaround may actually make sense. Wilcox expressed some skepticism on that score. 

Overall, there are a bunch of readahead problems to solve. "I'm going to say the 'f' word; [folios](<https://lwn.net/Articles/893512/>) play into this a bit." The use of larger folios is driven by readahead, currently; the larger the readahead data gets, the larger the folios to hold it get. That is useful for testing. Filesystems that support larger folios, which is only XFS for now (though AFS and v9fs patches are queued), will allocate four-page (i.e. order-2) folios. 

Wilcox said that French made him aware that Windows does huge reads for readahead on CIFS. French agreed with that, noting that because of the expected network latency, Windows can readahead up to 16MB, though that seems to degrade performance. He tested with a number of different sizes (256KB, 512KB, 1MB, 4MB) and found better performance with each of those, the most dramatic being seen when going to 512KB. On the Azure cloud, the value was set to 1MB because there were performance decreases for some workloads at 4MB. 

The Linux CIFS server defaults to 4MB, he said, based on the results of his testing. It is clear that anything less than 1MB performs worse unless there is a fast network in between. The problem he sees is how this value can get changed sanely, throttled or raised as appropriate. Sometimes the page cache knows more than the filesystem, or the reverse can be true, and the network layer needs to factor in as well. It is not clear to him how that can all be resolved. 

There is a mechanism to communicate this kind of information from the filesystem to the virtual filesystem (VFS) layer and page cache using the BDI ([`struct backing_dev_info`](<https://elixir.bootlin.com/linux/v5.18.3/source/include/linux/backing-dev-defs.h#L172>)), Wilcox said. That is where the VFS looks to find out the performance characteristics of the underlying storage. It may not currently have all of the right information, he said, but that's the place to put it. 

When user space is reading a file one byte at a time, a 64KB read is issued instead; a "was this readahead useful" marker is placed around 20KB into the buffer and when it is reached, another 64KB read is issued. The intent is that the second read completes before user space consumes the rest of the 44KB, but the filesystem has no idea of what the latency is for the read. One could imagine measuring how long it takes to do the read and comparing it with the user-space consumption rate to better determine when to schedule the next read, he said, but that is not done. 

That second 64KB read has its marker placed right at the beginning of the buffer; when that marker is reached, it decides to grow the readahead buffer and reads 128KB. It will increase once more (to 256KB), but not any further, though it should probably go up from there. The 256KB limit has been that way for 20 years and "I/O has hardly changed at all in that time", Wilcox said sarcastically; increasing that limit is many years overdue at this point. Josef Bacik read a chat comment from Jan Kara that said SUSE has had a limit of 512KB for years. Given that, Wilcox thought an immediate move to a 1MB maximum was in order. 

But, rather than increasing the size of the read directly, Wilcox would rather issue multiple 256KB reads back to back because that will help the page cache track if the data that is read ahead is actually being used. French said that would be preferable for CIFS, as multiple smaller reads are better performance-wise. Jeff Layton came in via Zoom to say that he thought a single, larger read would be better for NFS and was surprised to hear that smaller reads were better for CIFS. 

Howells said that which of the options is better is totally filesystem-dependent. Ted Ts'o said that it will also depend on the underlying storage; increasing the readahead size on memory-constrained devices may also be problematic. There is not going to be a single "magic readahead formula" that can be used in all situations. He suggested making it possible to experiment with different algorithms using BPF; that way a grad student, for example, could easily try out different ideas to see which worked best. Wilcox said that he had wanted to argue with that idea but then decided he liked it because it would make it someone else's problem, which was met with much laughter. 

Chuck Lever said that he thought the readahead situation was being greatly oversimplified in the discussion. In the original example of a program reading a byte at a time, there is no reason to increase beyond 64KB, since that will keep up with the program just fine. There are problems with requesting too much data and filling the page cache with pages that are not actually going to be read. 

Another problem is that queueing up a 1MB read, for example, on a network filesystem will hold up other smaller requests, like metadata updates, while the read is ongoing. He agrees with the need for experimentation and thinks that should take precedence over any immediate increase in the kernel's readahead size. There is a need for testing with a lot of different workloads, filesystems, and so forth to determine what the overall systemic effects of that kind of change would be.
