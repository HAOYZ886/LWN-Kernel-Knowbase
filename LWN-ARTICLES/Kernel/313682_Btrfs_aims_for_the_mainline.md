---
title: Btrfs aims for the mainline
url: https://lwn.net/Articles/313682/
date: "January 7, 2009"
category: "Btrfs; Filesystems-Btrfs; Locking mechanisms-Mutexes"
author: "By Jonathan Corbet January 7, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 7, 2009

The Btrfs filesystem has been under development for the last year or so; for much of that time, it has been widely regarded as the most likely "next generation filesystem" for Linux. But, before it can claim that title, Btrfs must stabilize and find its way into the mainline kernel. Btrfs developer Chris Mason has been saying for a while that he thinks the code will come together more quickly if it is merged relatively soon, even if it is not yet truly ready for production use. General experience with kernel development tends to support this position: in-tree code gets more review, testing, and fixes than out-of-tree code. So the development community as a whole has been reasonably supportive of a relatively early Btrfs merge. 

In [our last Btrfs episode](<http://lwn.net/Articles/302251/>), Andrew Morton suggested that a 2.6.29 merge be targeted. Chris would like that happen; to that end, he has [posted a version of Btrfs](<http://lwn.net/Articles/313343/>) for consideration. Unsurprisingly, that posting has already increased the amount of attention being paid to this code, with the result that Chris quickly got a list of things to fix. Most of those have now been addressed, but there are a few remaining issues which could still impede the merging of Btrfs in this development cycle. This article will look at the potential roadblocks. 

One of those is the user-space API. Btrfs brings with it a whole set of new `ioctl()` calls, none of which have been seriously reviewed or even documented. These calls perform functions like creating snapshots, initiating defragmentation, creating or resizing subvolumes, adding devices to the volume set, etc. Interestingly, there has been no real complaint about the volume-management features of Btrfs in general. But the interface to features like that needs close scrutiny; normally, user-space APIs cannot be broken once they are merged into the mainline. There has been some talk of making an exception for Btrfs, since there is little chance of systems becoming dependent on a specific interface before Btrfs is production-ready. 

Still, once distributions start shipping Btrfs tools - to help testers if nothing else - an API change would cause pain. Any potential for this kind of pain would make API changes very hard to do. So Linux may well end up being stuck with the early Btrfs API. Given that at least one developer [thinks that this API needs a serious rework](<https://lwn.net/Articles/313691/>), this issue could turn out to be a serious roadblock indeed. 

Then, there is the issue of the special-purpose locking primitives used in Btrfs. To understand this discussion, it's worth looking at the locking function used within Btrfs: 

```
int btrfs_tree_lock(struct extent_buffer *eb)
        {
    	int i;
    
    	if (mutex_trylock(&eb->mutex))
    	    return 0;
    	for (i = 0; i < 512; i++) {
    	    cpu_relax();
    	    if (mutex_trylock(&eb->mutex))
    		return 0;
    	}
    	cpu_relax();
    	mutex_lock_nested(&eb->mutex, BTRFS_MAX_LEVEL - btrfs_header_level(eb));
    	return 0;
        }
```

The lock in question is a mutex, but it is being acquired in an interesting way. If the lock is held by another process, this function will poll it up to 512 times, without sleeping, in the hope that it will become available quickly. Should that happen, the lock can be acquired without sleeping at all. After 512 unsuccessful attempts, the function will finally give up and go to sleep. 

Chris [justifies this behavior](<https://lwn.net/Articles/313686/>) this way: 

Btrfs is using mutexes to protect the btree blocks, and btree searching often hits hot nodes that are always in cache. For these nodes, the spinning is much faster, but btrfs also needs to be able to sleep with the locks held so it can read from the disk and do other complex operations. 

For btrfs, dbench 50 performance doubles with the unconditional spin, mostly because that workload is almost all in ram. For 50 procs creating 4k files in parallel, the spin is 30-50% faster. This workload is a mixture of disk bound and CPU bound. 

That kind of performance increase seems worth going for. In fact, it reflects a phenomenon which has been observed in other situations as well: even when sleeping locks are used, performance often improves if a processor spins for a while in the hope that a contended lock will become available. If the lock can be acquired without sleeping, then the overhead associated with putting the process to sleep and waking it up can be avoided. Beyond that, though, there is the fact that the process seeking to acquire the lock is probably well represented in the CPU's cache. Allowing that process to continue to run will, if the lock can be acquired quickly, almost certainly lead to better system performance. 

For this reason, the [adaptive realtime locks patch](<http://lwn.net/Articles/271817/>) was developed last year, though it never found its way into the mainline. In response to the Btrfs discussion, Peter Zijlstra proposed [a spinning mutex patch](<http://lwn.net/Articles/313604/>) which is intended to provide the same benefits as the special Btrfs locking function, but for more general use and without the addition of magic constants. In Peter's patch, an attempt to acquire a contended lock will spin for as long as the process holding that lock is actually running on a CPU. If the lock holder goes to sleep, any process trying to acquire the lock also goes to sleep. The heuristic seems to make sense, though detailed benchmarks have not been posted. The patch was received reasonably well, though Linus has [insisted](<https://lwn.net/Articles/313687/>) that some changes be made. 

So a more general spinning mutex may well find its way into the mainline. Whether it will go in for 2.6.29 is not clear, though. Developers tend to like their core locking primitives to be reasonably well tested; merging something which was developed toward the end of the merge window could be a hard sell. Until something like that happens, Chris is [uninterested in removing his special locking function](<https://lwn.net/Articles/313688/>): 

But, if anyone working on adaptive mutexes is looking for a coder, tester, use case, or benchmark for their locking scheme, my hand is up. Until then, this is my for loop, there are many like it, but this one is mine. 

Finally, there is the question of the name. Some reviewers have suggested that the filesystem should be merged with a name which makes it clear that it's not meant for production use - "btrfsdev," for example. Chris is resistant to that idea, noting that, unlike existing filesystems, Btrfs is known to be new and has no reputation for stability. He has stated his willingness to make the change, though, if it is truly considered to be necessary. Bruce Fields [pointed out](<https://lwn.net/Articles/313689/>) that calling it "Btrfs" from the beginning could possibly burn future developers who boot an old kernel (with a non-production Btrfs) after switching to a newer, production-ready version of the filesystem. 

All of this adds up to an uncertain fate for Btrfs in 2.6.29; there are a fair number of open issues and it's late in the merge window. Of course, Btrfs could be merged after 2.6.29-rc1; since it is a completely new subsystem, it won't cause regressions. But if Linus concludes that there are enough loose ends in the current Btrfs code, he may just decide to give it one more development cycle before bringing it into the mainline. So, while nobody seems to doubt that Btrfs will go in, the question of when remains open. 

(With any luck, we hope to have an authoritative article on Btrfs for this page in the near future, once the author - you know who you are! - gets it written. Stay tuned.)
