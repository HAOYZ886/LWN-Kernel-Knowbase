---
title: Documenting page flags by committee
url: https://lwn.net/Articles/974515/
date: "May 22, 2024"
category: "Documentation; Memory management-struct page"
author: "By Jonathan Corbet May 22, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

For every page of memory in the system, the kernel maintains a set of page flags describing how the page is used and various aspects of its current state. Space for page flags has been in [chronic short supply](<https://lwn.net/Articles/335768/>), leading to a desire to eliminate or consolidate them whenever possible. That objective, though, is hampered by the fact that the purpose of many page flags is not well understood. In a memory-management-track session at the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Matthew Wilcox set out to cooperatively update the page-flag documentation to improve that situation. 

Wilcox had no presentation to give; instead, he put up an editor window containing a new documentation file for page flags, then told the audience "shout at me, I'll write it down". The first flag to be covered was `Locked`; the text that resulted was: 

> This flag is per-folio. If you attempt to lock a page, you will lock the entire folio. The folio lock is used for many purposes. In the page cache, folios are locked before reads are started and unlocked once the read has completed. The folio is also locked before writeback starts; see the writeback flag for more detail. The truncation path takes the folio lock, and folios are also locked while being inserted into page tables in order to prevent races between truncation and page fault. 

These semantics, Wilcox said, are why the lockdep locking checker does not work with this flag; it is taken and released in different contexts, which lockdep cannot handle. 

The next flag was `Writeback`, which ended up being described as: 

> Per-folio. This is kind of a lock. We downgrade to it having taken the lock [`Locked`] flag. Released after writeback completes, but lock flag may be released any time after writeback flag set. Depends on filesystem whether needs to do more between. We can wait for writeback to complete by waiting on this flag. Folio put to tail of LRU for faster reclaim. 
> 
> Can prevent tearing write is filesystem needs stable folios. Truncate will wait for flag to clear. 

Clearly, there is some editing work yet to be done. 

For the `Dirty` flag, the result was: 

> Also set during buffered IO. Referenced first time, accessed second time. Used during reclaim to determine disposition (activate, reclaim, etc). At least one byte of the folio contents is newer than on disk and the writeback flag is not yet set. Folios may be both dirty and not uptodate. Lazyfree pages can drop the dirty bit. Dirty flag clear for file folios when we start writeback. Set dirty flag when removed from swapcache. If already dirty, folios can be mapped writable without notifying filesystem. Complicated interfaces to set, easy to get wrong. 

Jason Gunthorpe added that there are a lot of users of `get_user_pages()` that set this flag; all of them are wrong. 

For `Uptodate`: ""Every byte of the folio contents is at least as new as the contents of disk. Implicit write barrier"". In the room, it was suggested that some filesystems clear this bit when writeback fails, but others thought that perhaps this behavior had been removed. 

For the `LRU` flag, all that was said was: ""Folio has been added to the LRU and is no longer in percpu folio_batch"". The `Head` flag was described equally tersely as: ""This folio is a large folio. It is not set on order-0 folios"". The `Waiters` flag means: ""Page has waiters, check its waitqueue. Only used by core code. Don't touch"". For the `Active` flag: ""On the active LRU list. Can be set in advance to tell kernel to put it on the right list"". 

When it came to `Workingset`, it seemed that nobody really knows what this flag means. Wilcox wrote down: 

> Set on folios in pagecache once readahead pages actually accessed. Set on LRU pages that were activated have been deactivated, treat refault as thrashing. Refault handler also sets it on folios that were hot before reclaimed used by PSI computation. 

The `Referenced` flag means: 

> Per-folio flag. At least one page table entry has a accessed bit set for this folio. We set this during scan. Also set during buffered IO. Referenced first time, accessed second time. Used during reclaim to determine disposition (activate, reclaim, etc). 

The flag named `Owner_Priv_1` was described as: ""Owner use. If pagecache, fs may use Used as Checked flag by many filesystems. Used as SwapBacked flag by swap code"". The final flag discussed in the session was `Arch_1`, with this result: 

> Many different uses depending on architecture. Often used as a "dcache clean" or, confusingly as "dcache dirty". Check with your architecture. 
> 
> s390 uses it for basically everything. 
> 
> Historically was used on a per page basis. Think we've eliminated all per-page uses now so should only be set on folios. 

After the session, Wilcox [posted the result](<https://lwn.net/ml/linux-mm/ZkOu4yXP-sGGtwc4@casper.infradead.org/>) on the linux-mm mailing list, where there have been a couple of follow-on comments. Whether this kind of whole-room documentation authoring will (or should) catch on remains to be seen; the information that was captured is more than was available before, but one might be forgiven for concluding that the use of these flags remains obscure for almost everybody.
