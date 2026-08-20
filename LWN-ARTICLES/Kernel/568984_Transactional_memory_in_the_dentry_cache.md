---
title: Transactional memory in the dentry cache
url: https://lwn.net/Articles/568984/
date: "October 2, 2013"
category: Transactional memory
author: "By Jonathan Corbet October 2, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 2, 2013

LWN recently [described the "lockref" mechanism](<https://lwn.net/Articles/565734/>) merged into the 3.12 kernel. Lockrefs aim to reduce locking overhead in situations where all that needs to happen is the modification of a reference count, but the data structure as a whole must be held invariant while that count is changed. This new locking primitive has helped to reduce the overhead of locking in the system's dentry cache — the extensive data structure that caches mappings between file names and inodes. But dentry cache overhead strongly affects the performance of the system as a whole, so it is not surprising that there is a desire to improve things even more; that appears to be especially true in the 3.12 development cycle, which has seen a lot of attention paid to the reduction of core locking overhead. 

One might think that it would be hard to improve on the performance of a lockref in this area, but there is one technology that might just help in this regard: transactional memory. For some years now, transactional memory has been one of those hardware features that was destined to solve a number of our scalability problems. As is often the case, that future has been slow to arrive in the real world. But some manufacturers are now shipping CPUs with some transactional memory support built into them, and software is starting to take advantage of this feature. See, for example, [this article](<https://lwn.net/Articles/534758/>) describing the work done to add transactional memory support to the GNU C library. 

Linus recently got an itch to upgrade to a newer, faster desktop computer; when he chose his new processor, he made a point of getting one that provided transactional memory support. So, he decided, the time had come to try to use that support to further speed dentry cache locking. The result was [a short, proof-of-concept patch](<https://lwn.net/Articles/568962/>) that shows how things could work. The core part of the patch is worth showing directly: 

```
asm goto("xbegin %l[repeat]": : :"memory","ax":repeat);
        if (unlikely(d_unhashed(dentry)))
        	goto xabort;
        if (unlikely(!is_simple_dput(dentry)))
        	goto xabort;
        if (unlikely(!arch_spin_value_unlocked(dentry->d_lock.rlock.raw_lock)))
        	goto xabort;
        dentry->d_lockref.count--;
        asm volatile("xend");
        return;
    
    xabort:
        asm volatile("xabort $0");
    
    repeat:
        /* Fallback to lockref code */
```

The first line adds the `xbegin` instruction that starts the transaction. This instruction behaves a little strangely, in that its influence extends over the code that follows. Execution will continue with the code after the `xbegin`, but, should the transaction abort for any reason, all changes made during the transaction will be rolled back and control will jump to the address provided to `xbegin` (the label `repeat` in this case). 

What follows is a set of tests to determine whether it is truly safe to decrement the reference count without holding the dentry spinlock. In particular, the dentry must be hashed (have a name associated with it, essentially), be on the LRU list, and not be locked. Reading these fields of the `dentry` structure adds them to the transaction; should some other code make a change to one of them, the transaction will be aborted by the processor. If any of the tests show that the dentry is not in a suitable state for a quick reference count decrement, the code uses the `xabort` instruction to make the transaction fail. Otherwise, the reference count will be decremented and the `xend` instruction will bring the transaction to a successful conclusion. The reference count, too, will become part of the transaction; should any other processor also try to change it, the transaction will abort. 

The code as written is clearly not intended for merging; one does not put late-model x86 assembly code directly into generic filesystem code, after all. But it is enough to get a sense for how well transactional memory works in this situation. According to Linus, ""profiles with this look beautiful"", with the locking cost associated with the `dput()` operation reduced essentially to zero. So there may well be a place for transactional memory within the dentry cache. 

Whether this technique will see much wider use in the kernel remains to be seen, though. Andi Kleen posted [a patch using transactional memory for most kernel locks](<https://lwn.net/Articles/544125/>) back in March, but that work did not go very far, mostly because he could not provide any benchmark numbers showing what kind of performance improvements could be expected. In his patch, Linus made it clear that he suspects those improvements will be small, saying ""I doubt the intel TSX patches are going to be useful (if they were, Intel would be crowing about performance numbers now that the CPU's are out, and they aren't)."" Instead, he has suggested that this feature will only be useful in a few places: 

Quite frankly, from all I've seen so far, the kernel is not going to have very good luck with things like lock elision, because we're really fine-grained already, and at least the Intel lock-elision (don't know about POWER8) basically requires software to do prediction on whether the transaction will succeed or not, dynamically based on aborts etc. And quite frankly, by the time you have to do things like that, you've already lost. We're better off just using our normal locks. 

So as far as I'm concerned, transactional memory is going to be useful - *if* it is useful - only for specialized code. Some of that might be architecture-internal lock implementations, other things might be exactly the dput() kind of situation. 

The end result is that Andi's patches may be a good starting point for transactional memory support in a more architecture-independent way, but we may not see transactional memory used as a general locking mechanism (or lock-elision mechanism) in the kernel. 

Along the lines of architecture-independence, it is worth noting that Intel was not the first to ship a processor with transactional memory support; the PowerPC architecture has had similar functionality for a couple of years. Naturally, the feature works a little differently in that architecture, but the basic functionality is the same. So it should be possible to create some sort of wrappers around transactional memory that can be used in common code. That is, if kernel-space transactional memory can be made to work at all in an efficient manner on PowerPC; there are [some challenges](<https://lwn.net/Articles/568989/>) in the way of getting that functionality working reliably. 

What one might conclude is that, while transactional memory is a useful technology for speeding up specific bits of high-profile kernel code, it does not look like a general solution to the locking problem at this point. That could change in the future, of course, as hardware transactional memory gets faster and more capable. But, for now, transactional memory looks mostly like a useful trick to squeeze a little more performance out of a few highly optimized code paths.
