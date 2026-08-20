---
title: Smarter  write throttling
url: https://lwn.net/Articles/245600/
date: "August 14, 2007"
category: "Memory management-Writeout throttling; Write throttling"
author: "By Jonathan Corbet August 14, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 14, 2007

Whenever a process performs a normal, buffered `write()` to a file, it ends up creating one or more dirty pages in memory. Those pages must eventually be written to disk. Until the data moves to persistent storage, the pages of memory it occupies cannot be used for any other purpose, even if the original writing process, as is often the case, no longer needs them. It is important to prevent dirty pages from filling too much of the system's memory; should the dirty pages take over, the system will find itself under severe memory pressure, and may not even have enough memory to perform the necessary writes and free more pages. Avoiding this situation is not entirely easy, though. 

As a general rule, software can create dirty pages more quickly than storage devices can absorb them. So various mechanisms must be put in place to keep the number of dirty pages at a manageable level. One of those mechanisms is a simple form of write throttling. Whenever a process dirties some pages, the kernel checks to see if the total number of dirty pages in the system has gotten too high. If so, the offending process is forced to do some community service by writing pages to disk for a while. Throttling things in this way has two useful effects: dirty pages get written to disk (and thus cleaned), and the process stops making more dirty pages for a little while. 

This mechanism is not perfect, however. The process which gets snared by the global dirty pages threshold may not be the one which actually dirtied most of those pages; in this case, the innocent process gets put to work while the real culprit continues making messes. If the bulk of the dirty pages must all be written to a single device, it might not be beneficial to throttle processes working with files on other disks - the result could be that traffic for one disk essentially starves the others which could, otherwise, be performing useful work. Overall, the use of a single global threshold can lead to significant starvation of both processes and devices. 

It can get worse than that, even. Consider what happens when block devices are stacked - a simple LVM or MD device built on top of one or more physical drives, for example. A lot of I/O through the LVM level could create large numbers of dirty pages destined for the physical device. Should things hit the dirty thresholds at the LVM level, however, the process could block before the physical drive starts writeback. In the worst case, the end result here is a hard deadlock of the system - and that is not generally the sort of reliability that users expect of their systems. 

Peter Zijlstra has been working on a solution in the form of the [per-device write throttling patch set](<http://lwn.net/Articles/244443/>). The core idea is quite simple: rather than use a single, global dirty threshold, each backing device gets its own threshold. Whenever pages are dirtied, the number of dirty pages which are destined for the same device is examined, and the process is throttled if its specific device has too many dirty pages outstanding. No single device, then, is allowed to be the destination for too large a proportion of the dirty pages. 

Determining what "too large" is can be a bit of a challenge, though. One could just divide the global limit equally among all block devices on the system, but the end result would be far from optimal. Some devices may have a great deal of activity on them at any given time, while others are idle. One device might be a local, high-speed disk, while another is NFS-mounted over a GPRS link. In either case, one can easily argue that the system will perform better if the faster, more heavily-used devices get a larger share of memory than slow, idle devices. 

To make things work that way, Peter has created a ["floating proportions" library](<https://lwn.net/Articles/245603/>). In an efficient, mostly per-CPU manner, this library can track events by source and answer questions about what percentage of the total is coming from each source. In the writeback throttling patch, this library is used to count the number of page writeback _completions_ coming from each device. So devices which are able to complete writeback more quickly will get a larger portion of the dirty-page quota. Devices which are generally more active will also have a higher threshold. 

The patch as described so far still does not solve the problem of one user filling memory with dirty pages to the exclusion of others - especially if users are contending for the bandwidth of a single device. There is another part of the patch, however, which tries to address this issue. A different set of proportion counters is used to track how many pages are being dirtied by each task. When a page is dirtied and the system goes to calculate the dirty threshold for the associated device, that threshold is reduced proportionately to the task's contribution to the pile of dirty pages. So a process which is producing large numbers of dirty pages will be throttled sooner than other processes which are more restrained. 

This patch is in its eighth revision, and there has not been a lot of criticism this time around. Linus's [response](<https://lwn.net/Articles/245610/>) was: 

Ok, the patches certainly look pretty enough, and you fixed the only thing I complained about last time (naming), so as far as I'm concerned it's now just a matter of whether it *works* or not. I guess being in -mm will help somewhat, but it would be good to have people with several disks etc actively test this out. 

The number of reports so far has been small, but some testers have said that this patch makes their systems work better. It was recently removed from -mm "due to crashiness," though, so there are some nagging issues to be taken care of yet. In the longer term, the chances of it getting in could be said to be fairly good - but, with memory management patches like this, one never knows for sure.
